# 晒阳 — 架构设计文档

> 版本：V3 | 2026-07-21
> 类型：纯工作流型工具，非自主Agent
> 定位：输入Hermes对话session → 按六角度拆解ReAct循环 + 成本透视 → 输出Word分析报告
> 变更：V3新增成本透视——步骤级token消耗、效率指标、成本收益分析

---

## 一、项目定位

晒阳是一个确定性工作流工具。用户说"react循环分析"，它读Hermes的state.db，解析指定session的对话，按`Agent = LLM + [上下文 + 工具 + 约束 + 验证 + 纠正]`六角度拆解每轮ReAct行为，生成Word分析报告。

核心原则：能确定就确定，能不用Agent就不用Agent。LLM仅在语义分类和叙述生成两处介入。所有确定性步骤中间结果落盘，支持断点续跑。

**双轨分析**：结构透明（六角度——每一步是什么、为什么、怎么验证纠正）+ 成本透明（token消耗——每一步烧了多少、效率如何、哪步最贵）。数据来源：state.db的messages表和sessions表已有完整的token统计字段，无需额外埋点。成本数据走确定性计算（parser层），不额外消耗LLM调用。

---

## 二、架构总览

```
state.db (SQLite)
    │
    ▼
┌─────────────┐
│  reader     │  读session元信息 + messages全字段（23字段，重点reasoning/system_prompt等）
└──────┬──────┘
       │  session_meta + messages（含reasoning, tool_call_id, tool_name, finish_reason, token_count）
       ▼
┌─────────────┐
│  parser     │  纯确定性：轮次边界识别、工具调用名/时间戳提取。不做语义理解
└──────┬──────┘
       │  turns [{turn_id, user_content, assistant_actions[{type, tool_name, args, timestamp}], ...}]
       ▼
┌─────────────┐
│  classifier │  规则层: 工具→工具角度, 技能→上下文角度, system_prompt→约束角度, 用户纠正→纠正角度
│             │  LLM层:  将全部六角度的语义分类打包一次LLM调用（含few-shot示例+JSON schema）
└──────┬──────┘
       │  angles + .cache/03_classifier.json
       ▼
┌─────────────┐
│  narrator   │  LLM调用: 基于分类结果+原始数据生成分析叙述（JSON schema约束输出格式）
└──────┬──────┘
       │  narrative + .cache/04_narrator.json
       ▼
┌─────────────┐
│  writer     │  模板填充 → python-docx生成Word（排版函数内置，不跨项目依赖）
└──────┬──────┘
       │
       ▼
  React循环透明化分析报告.docx
```

> 虚线框 `.cache/` 为中间结果持久化点，支持断点续跑。

---

## 三、模块设计

### 3.1 reader.py — 会话读取

**职责**：从Hermes的state.db读取指定session的完整数据——session元信息和messages全字段。

**数据源**：实际state.db有两张核心表：

- **sessions表**（38字段）：读取 `session_id, system_prompt, model, input_tokens, output_tokens, reasoning_tokens, estimated_cost_usd, message_count, tool_call_count, parent_session_id`
- **messages表**（23字段）：读取 `id, session_id, role, content, reasoning, reasoning_content, tool_calls, tool_call_id, tool_name, finish_reason, token_count, timestamp`

**输入**：
- state.db路径（可配置，默认从.env读取）
- session_id（用户指定，或自动取最近一次session）

**输出**：
```python
{
    "session_meta": {
        "session_id": "20260721_...",
        "system_prompt": "You run on Hermes Agent...",  # ← 约束角度的核心数据源
        "model": "deepseek-v4-pro",
        "input_tokens": 45000,
        "output_tokens": 12000,
        "reasoning_tokens": 0,
        "estimated_cost_usd": 0.08,
        "message_count": 45,
        "tool_call_count": 15,
        "parent_session_id": None
    },
    "messages": [
        {
            "role": "user" | "assistant" | "tool",
            "content": "...",
            "reasoning": "...",           # ← LLM推理链（LLM角度的核心数据源）
            "reasoning_content": "...",
            "tool_calls": [...],          # assistant消息可能包含
            "tool_call_id": "...",        # tool消息关联到哪个调用
            "tool_name": "...",           # 工具名称
            "finish_reason": "stop",      # ← LLM调用是否正常结束（可靠性角度）
            "token_count": 350,
            "timestamp": 1783079599.912
        }
    ]
}
```

**实现要点**：
- 用sqlite3直读，SQL参数化防注入
- 先查sessions表取meta，再查messages表取消息
- tool_calls字段是JSON字符串，需json.loads解析
- 按timestamp升序排列

---

### 3.2 parser.py — 对话解析

**职责**：纯确定性解析。识别对话轮次边界，提取工具调用的名称和时间戳。不做任何语义理解——`purpose`和`result_summary`不在此层生成。

**输入**：reader输出的messages列表

**输出**：
```python
{
    "turns": [
        {
            "turn_id": 1,
            "user_content": "如基于规则的正则过滤是什么意思",
            "user_timestamp": 1783079500.0,
            "assistant_actions": [
                # 每条是消息级别的原始数据，不含语义标注
                {"type": "llm_call", "reasoning": "...", "finish_reason": "stop"},
                {"type": "tool_call", "tool_name": "skill_view", "args": {"name": "thinking-foundation"}},
                {"type": "tool_call", "tool_name": "session_search", "args": {"query": "正则过滤"}},
                {"type": "response", "content": "这是你奶龙桌宠项目..."}
            ],
            "parallel_tool_calls": True,   # 基于timestamp间隔<1s判断
            "tool_count": 3,
            "has_user_correction": False   # 本user消息是否包含纠正意图
        }
    ],
    "total_turns": 9,
    "total_tool_calls": 15,
    "cost_breakdown": {
        "session_total": {"input_tokens": 45000, "output_tokens": 12000, "estimated_cost_usd": 0.08},
        "by_turn": [
            {
                "turn": 1,
                "effective_tokens": 800,     # 用户输入+Agent回答正文
                "overhead_tokens": 12000,    # 技能加载+工具调用+系统提示词
                "tool_cost": 0.002,          # 工具调用消耗的估算成本
                "is_parallel": True,
                "waste_tokens": 0,           # 失败重试浪费的token
                "efficiency": "low"          # 效率评级: high/medium/low/high_with_savings
            }
        ],
        "hotspots": [                        # 成本热点
            {"turn": 2, "cost": 0.025, "reason": "PDF提取+5次串行工具调用"},
            {"turn": 3, "cost": 0.015, "reason": "delegate_task子agent审查"}
        ],
        "efficiency": {
            "parallel_savings_estimate": 0.005,  # 并行调用节省的估算成本
            "waste_from_retries": 0.003,          # 失败重试浪费的成本
            "overhead_ratio": 0.76                 # 开销token占比(76%=技能+工具/总token)
        }
    },
    "estimated_total_tokens": 42000
}
```

**实现要点**：
- 按user→assistant配对识别轮次边界（user消息后紧跟的assistant消息属于同一轮）
- 从assistant的tool_calls JSON中提取tool_name和args
- 并行判断：同一assistant消息中相邻tool_call的timestamp间隔<1秒
- 用户纠正检测：user消息匹配关键词（"不对""再看看""8,9,10我回答了"）→标记has_user_correction
- 从session_meta.estimated_cost_usd和message token_count估算总token

**成本数据计算（确定性，不调LLM）**：
- 逐轮汇总：累加该轮所有messages的token_count → per-turn total
- 有效token：用户消息token_count + assistant回复正文token_count（排除tool_calls部分）
- 开销token：per-turn total - 有效token
- 工具成本：tool角色的messages的token_count之和
- 浪费token：finish_reason!="stop"的消息 + 同一工具连续2次以上调用的重复token
- 并行节省估算：并行轮次的总token × 0.3（非实测——对话未以串行方式发生过，基于静态前缀去重的推理估算）
- 开销占比：开销token / 总token
- 效率评级：high（开销占比<30%或并行节省显著）/ medium（正常范围）/ low（开销占比>70%或存在浪费token）/ high_with_savings（存在显著节省决策，如patch替代write_file）

成本数据随parser输出流入narrator。narrator在第一块全流程中只对效率评级为low或high_with_savings的步骤嵌入成本标注；在第三块成本透视中展示全部数据。

---

### 3.3 classifier.py — 六角度分类

**职责**：将每轮的行为按六角度归类。两层——规则层做确定性映射，LLM层做全六角度语义分类。

**规则层（不调LLM，确定性）**：

| 数据特征 | 归入角度 | 来源字段 |
|----------|----------|----------|
| tool_calls中存在tool_name | 工具 | messages.tool_calls |
| 调用了skill_view/skills_list | 上下文 | messages.tool_calls[].tool_name |
| session_meta.system_prompt | 约束 | sessions.system_prompt |
| 用户消息has_user_correction=True | 纠正 | parser输出 |
| finish_reason != "stop" | 可靠性（LLM异常终止） | messages.finish_reason |

**LLM层（一次调用处理所有轮次）**：

将全部轮次的对话原文+规则层分类+结构化数据（reasoning/system_prompt/finish_reason/tool_call记录）打包成一次prompt，要求LLM从六个角度识别：

- LLM角度：什么时候调了LLM、推理链（reasoning字段）体现了什么内部思考、做了什么决策、什么时候走了规则没调LLM
- 上下文角度：注入了什么技能/memory、用了什么已有知识、上下文缺失点在哪
- 工具角度：调了什么工具、为什么选它不选别的、工具结果怎么用、工具失败怎么处理（基于tool_name+finish_reason+回复内容）
- 约束角度：什么规则在限制行为（基于system_prompt+memory+回复中体现的约束执行效果）
- 验证角度：Agent怎么检查自己做得对不对、检查了什么
- 纠正角度：错了怎么补救、用户纠正后Agent如何调整

**输出（JSON schema约束）**：
```python
{
    "by_angle": {
        "LLM": [
            {"turn": 1, "action": "概念解释任务，判断为高确定性→直接回答不调工具"},
            {"turn": 2, "action": "复杂提取任务→选pymupdf→装依赖→提取→映射"}
        ],
        "上下文": [{"turn": 1, "action": "注入thinking-foundation和learning-profile, session_search获取chouxiang历史"}],
        "工具": [{"turn": 1, "action": "skill_view×2 + session_search, 共3次, 并行调用"}],
        "约束": [{"turn": 2, "action": "不做应声虫约束生效——用户回答有问题直接指出"}],
        "验证": [{"turn": 2, "action": "自检: 是否每道题都覆盖了全部子问题"}],
        "纠正": [{"turn": 3, "action": "用户指出Q8-Q10漏判→重新全页扫描→信任用户输入→更新评分"}]
    },
    "by_turn": [
        {
            "turn": 1,
            "angles": {
                "LLM": "概念解释，判断为高确定性→直接回答，不调工具",
                "上下文": "注入thinking-foundation+learning-profile, session_search获取chouxiang项目架构",
                "工具": "skill_view×2 + session_search, 共3次并行调用, 全部成功",
                "约束": "简洁直接高信息密度, 用生活化类比, 度原则(不灌太多)",
                "验证": "自检: 是否一句说清+有例子+关联用户知识",
                "纠正": null
            }
        }
    ]
}
```

**LLM prompt设计**（含few-shot和JSON schema）：

Prompt结构：
1. 角色定义：你是ReAct循环分析器
2. 六角度定义（逐角度说明"这个角度要分析什么，信息来源是哪些字段"）
3. 输入数据格式说明（标注每个字段对应哪个角度）
4. 2个few-shot示例（取自此前的样本报告）
5. 强制JSON输出schema
6. 当前session数据

6个角度在prompt中各自有信息来源映射：

| 角度 | prompt中提供的信息 |
|------|-------------------|
| LLM | reasoning字段原文, finish_reason, 回复内容, 工具调用选择逻辑 |
| 上下文 | 工具调用列表(筛选skill_view/session_search/memory), 回复中引用的已有知识 |
| 工具 | 完整tool_calls序列, tool_name, args, 并行/串行标记, 工具失败记录 |
| 约束 | system_prompt原文(截取前3000字符), memory内容, 回复中的约束执行证据 |
| 验证 | 回复中的自检语句, 用户纠正的触发时机 |
| 纠正 | has_user_correction标记, 用户纠正内容, 下一轮的行为变化 |

---

### 3.4 narrator.py — 叙述生成

**职责**：基于分类结果+原始数据，调用LLM生成分析叙述。输出JSON结构化数据供writer填充。

**输入**：
- classifier的分类结果（by_angle + by_turn）
- parser的原始轮次数据
- reader的session_meta

**输出**（强制JSON schema）：
```json
{
  "overview": "本次对话共9轮，涵盖三类主题：概念解释、思考题批改与存档、Agent透明化需求。共15次工具调用，2次用户纠正。",
  "turn_by_turn": [
    {
      "turn": 1,
      "title": "正则过滤概念解释",
      "narrative": "用户问'基于规则的正则过滤'的含义。Agent加载地基技能后...",
      "angles": {
        "LLM": "概念解释任务，判断为高确定性→直接回答不调工具。采用了类比法(机场安检金属探测门)。",
        "上下文": "注入thinking-foundation和learning-profile，session_search确认用户知识背景。",
        "工具": "skill_view×2 + session_search，共3次并行调用，全部成功。",
        "约束": "度原则(不灌太多)、生活化类比、不做应声虫。",
        "验证": "自检：是否一句说清+有例子+关联用户知识。",
        "纠正": null
      }
    }
  ],
  "angle_summary": {
    "LLM": "每轮用户消息都触发LLM调用，共9次。决策类型分布：概念解释4次、内容提取1次、格式化输出1次、需求审查1次、讨论澄清2次。没有跳过LLM直接走规则的情况。",
    "上下文": "固定注入thinking-foundation+learning-profile+memory。动态累积对话上下文。关键缺失：第2轮PDF注释Y坐标映射依赖工具输出而非验证结果。",
    "工具": "15次工具调用，skill_view占比最高(5/15)。并行调用模式在第1轮使用1次(3工具并行)。串行调用5次受pymupdf安装依赖限制。工具异常1次(pymupdf venv未安装)。",
    "约束": "7条活跃约束：角色定义、地基绑定、不做应声虫、度原则、铁律(不擅自执行)、文件操作确认、决策不可再议。约束执行有效——第8轮用户说'先别做'，Agent只审查未执行。",
    "验证": "当前验证全部为自检，无外部验证。第2轮注释提取的验证缺失导致Q8-Q10误判。验证改进方向：对确定性操作增加工具层验证(如提取后统计数量)。",
    "纠正": "2次纠正。纠正1: 用户指出Q8-Q10误判→重新扫描→信任用户输入→更新评分。纠正2: 用户说'API是格式，内容填环境变量'→用三层模型重新解释。纠正依赖用户信号或LLM内部判断，无主动纠正检测。"
  },
  "findings": [
    "发现1: 决策不可见——用户看到工具调用记录但看不到为什么选工具A不选B",
    "发现2: 验证不可见——每轮回答后的自检过程用户看不到",
    "发现3: 纠正不可见——用户纠正后Agent内部如何调整判断逻辑用户看不到"
  ],
  "improvements": [
    "改进方向1: 事前声明——每轮工具调用前声明目的(层次一)",
    "改进方向2: 决策日志——复杂任务末尾附决策轨迹(层次三)",
    "改进方向3: 六角度标注——在决策日志中标注每步行为归属哪个角度"
  ],
  "cost_analysis": {
    "overview": "本次对话共消耗57,000 tokens（输入45K+输出12K），估算成本$0.08。开销token占比76%，工具调用是主要成本来源。",
    "per_turn_table": [
      {"turn": 1, "tokens": 12800, "cost": 0.012, "main_driver": "3次技能加载(并行)", "efficiency": "high"},
      {"turn": 2, "tokens": 18500, "cost": 0.025, "main_driver": "5次串行工具调用+pymupdf安装", "efficiency": "low"}
    ],
    "hotspots": [
      "第2轮是最贵的单轮($0.025)，占全对话31%。原因是pymupdf安装+5次串行工具调用无法并行化。",
      "第3轮子agent审查($0.015)是额外成本，但发现了9个问题，避免了后续更大返工。ROI为正。"
    ],
    "efficiency_analysis": {
      "parallel_turns": [1, 8],
      "serial_turns": [2, 3, 4, 5, 6, 7, 9, 10],
      "waste_events": [
        {"turn": 2, "event": "pymupdf venv未安装→切换Python重试", "wasted_tokens": 2000, "wasted_cost": 0.002}
      ],
      "overhead_breakdown": {
        "system_prompt": 3000,
        "skills_loading": 15000,
        "tool_calls": 12000,
        "reasoning": 3000
      }
    },
    "findings": [
      "技能加载占总token的26%，其中thinking-foundation+learning-profile每轮都重复注入。如果KV Cache机制正常运转，这些应该被缓存而非每次重算。",
      "工具调用串行化是成本放大器——第2轮的5次串行调用使该轮成本飙升。并行化可节省约30%。",
      "delegate_task的成本($0.015)在本次对话中的ROI为正——发现的9个问题避免了后续架构返工。但审查成本是固定的，对小修改ROI可能为负。"
    ]
  }
}
```

**LLM prompt设计**：

Prompt包含：
1. 严格的JSON输出格式要求——`只输出JSON，不要解释，不要markdown代码块包裹`
2. 完整输出schema（含cost_analysis字段）
3. 1个完整示例（取自样本报告）
4. 当前session的分类结果+原始数据+成本数据注入

**成本分析的LLM指令**（narrator prompt中追加）：
```
基于提供的per-turn token数据，生成成本分析：
1. overview：一句话总结总消耗
2. per_turn_table：每轮的成本、主要驱动因素、效率评级(high/medium/low)
3. hotspots：最贵的2-3轮，解释为什么贵
4. efficiency_analysis：并行vs串行对比、浪费事件、开销分解
5. findings：3个成本优化建议，基于数据而非猜测

注意：成本数据来自state.db的messages.token_count和sessions.estimated_cost_usd，是真实数据。不要编造数字。
```

---

### 3.5 writer.py — Word生成

**职责**：模板化生成Word文档。排版函数内置于本模块，不跨项目依赖。

**实现**：
- 从project-analyzer的`write_word_report()`拷贝`_set_run_font()`、表格生成、markdown清理等核心函数到本模块内
- 源文件路径注明在代码注释中：`# 源自 project-analyzer/orchestrator.py`
- 章节结构固定：
  - 封面（标题 + session元信息：日期/模型/token消耗/消息数）
  - 一、对话概览
  - 二、逐轮ReAct还原（每轮一页，按六角度展开）
  - 三、六角度综合分析
  - 四、成本透视（per-turn成本表、热点分析、效率指标、开销分解、优化建议）
  - 五、核心发现与改进方向
  - 六、附录：工具调用时间线表格
- 排版规范：
  - 中文楷体12pt，英文Times New Roman
  - 标题黑体加粗黑色（一级16pt/二级14pt/三级12pt）
  - 表格9pt楷体，表头加粗
  - 每章单起一页
  - 段间不空行，行距1.3倍
  - 所有markdown符号清除

---

### 3.6 main.py — 工作流编排

**职责**：串联所有模块，支持断点续跑和大session分块。

```python
def run(session_id=None, db_path=None, output_dir=None, no_cache=False):
    cache_dir = Path(output_dir) / ".cache"
    
    # ── 1. 读 ──
    if not no_cache and (cache_dir / "01_reader.json").exists():
        session_data = json.loads((cache_dir / "01_reader.json").read_text())
    else:
        session_data = reader.read(db_path, session_id)
        _save_cache(cache_dir, "01_reader.json", session_data)
    
    # 估算token，超阈值自动分块
    chunk_mode = _should_chunk(session_data)  # >50轮 或 >30000 token
    
    # ── 2. 解析 ──
    if not no_cache and (cache_dir / "02_parser.json").exists():
        parsed = json.loads((cache_dir / "02_parser.json").read_text())
    else:
        parsed = parser.parse(session_data)
        _save_cache(cache_dir, "02_parser.json", parsed)
    
    # ── 3. 分类 ──
    if not no_cache and (cache_dir / "03_classifier.json").exists():
        classified = json.loads((cache_dir / "03_classifier.json").read_text())
    else:
        if chunk_mode:
            classified = classifier.classify_chunked(parsed)  # 每20轮一组
        else:
            classified = classifier.classify(parsed)
        _save_cache(cache_dir, "03_classifier.json", classified)
    
    # ── 4. 叙述 ──
    if not no_cache and (cache_dir / "04_narrator.json").exists():
        narrative = json.loads((cache_dir / "04_narrator.json").read_text())
    else:
        narrative = narrator.narrate(classified, parsed, session_data["session_meta"])
        _save_cache(cache_dir, "04_narrator.json", narrative)
    
    # ── 5. 输出 ──
    report_path = writer.write(narrative, session_data["session_meta"], output_dir)
    
    return report_path
```

无循环、无分支、无Agent自主决策。LLM失败的降级策略见第六章。

---

## 四、数据流

```
sessions表 ──┐
             ├──→ [reader] ──→ session_meta + messages(全23字段)
messages表 ──┘                        │
                                      ▼
                                 [parser]
                                      │
                              turns (含parallel标记, has_user_correction)
                                      │
                    ┌─────────────────┼─────────────────┐
                    │ 规则层           │  LLM层          │
                    │ 工具→工具角度    │  全六角度语义    │
                    │ 技能→上下文角度  │  分类(few-shot   │
                    │ sys_prompt→约束  │  +JSON schema)  │
                    │ 纠正→纠正角度    │                 │
                    └────────┬────────┘                 │
                             │                          │
                         angles                     .cache落盘
                             │
                    ┌────────┴────────┐
                    │  [narrator] ← LLM (JSON schema约束输出)
                    │       │
                    │  narrative + .cache落盘
                    └────────┬────────┘
                             │
                         [writer]
                             │
                     report.docx
```

> LLM调用失败时的降级路径：规则层结果 → narrator用简化prompt（只要求结构不要求深度） → 报告中标注"本次分析未包含LLM语义层分类"。

---

## 五、LLM调用可靠性设计

### 5.1 LLM调用清单

一次完整运行最多2次LLM调用：

| 调用 | 模块 | 作用 | 输入大小 | 失败后果 |
|------|------|------|----------|----------|
| 调用1 | classifier | 全六角度语义分类 | 全部对话+规则结果 | 语义标注丢失 |
| 调用2 | narrator | 生成分析叙述 | 分类结果+原始数据 | 无报告输出 |

### 5.2 每次调用的保护机制

```
调用LLM:
├── 超时: 60秒
├── 重试: 3次，指数退避(2s→4s→8s)
├── 失败处理:
│   ├── 调用1失败 → 规则层分类结果直接传给narrator
│   │               narrator在prompt中收到标注"无LLM语义分类"
│   │               报告中标注"本次分析未包含LLM语义层分类"
│   └── 调用2失败 → 从.cache/04_narrator.json恢复
│                   若缓存也不存在 → 输出规则层分类的纯数据报告(无叙述文本)
└── 重试耗尽 → 按上述降级路径
```

### 5.3 中间结果持久化

每步完成后立即写入`output_dir/.cache/{step_name}.json`：

```
output_dir/
├── .cache/
│   ├── 01_reader.json       # reader输出
│   ├── 02_parser.json       # parser输出
│   ├── 03_classifier.json   # classifier输出
│   └── 04_narrator.json     # narrator输出
└── React循环透明化分析报告.docx
```

- main.py启动时检查缓存，跳过已完成步骤
- 传`--no-cache`强制重跑全部
- 缓存文件带session_id前缀，避免不同session互相覆盖

---

## 六、大Session处理策略

**问题**：用户可能分析50+轮的session，单次LLM prompt会超出上下文窗口（DeepSeek ~128K tokens）。

**检测**：parser完成后计算`estimated_total_tokens`：
- 消息总字符数 ÷ 4（中英文混合估算）
- 加上prompt模板的固定开销（~2000 tokens）

**分块策略**（超过30000 token或超过50轮时触发）：
1. 每20轮一组，分组独立调用classifier的LLM层（每组有自己的few-shot上下文）
2. 所有组的分类结果合并传给narrator
3. narrator收到的是摘要级数据（每组的六角度分类结果），而非全部原文
4. 报告中标注"本session共X轮，已分块分析"

**分块不影响管道结构**：parser→classifier的接口不变，classifier内部判断是否分块。

---

## 七、技术选型

| 组件 | 选型 | 理由 |
|------|------|------|
| 数据库读取 | sqlite3（标准库） | state.db就是SQLite |
| LLM调用 | requests + DeepSeek API | 现有依赖，零额外成本 |
| Word生成 | python-docx | 排版函数内置，无跨项目依赖 |
| 配置管理 | .env文件 + python-dotenv | 项目级隔离 |
| 运行方式 | CLI：python main.py [session_id] | 简单直接 |

---

## 八、配置

.env文件：
```
STATE_DB_PATH=C:\Users\Luo\AppData\Local\hermes\profiles\learning\state.db
DEEPSEEK_API_KEY=sk-xxx
OUTPUT_DIR=E:\work\word\study\react分析报告
```

开源时.env.example不含密钥：
```
STATE_DB_PATH=/path/to/hermes/profiles/learning/state.db
DEEPSEEK_API_KEY=your-deepseek-api-key
OUTPUT_DIR=./output
```

---

## 九、与Hermes的集成方式

触发方式：用户在Hermes中说"react循环分析" → Hermes执行 `python E:\work\word\study\晒阳\main.py [session_id]`

不传session_id时自动取最近一次session。

无需Hermes插件、无需MCP、无需API回调。就是一个独立的CLI工具，从state.db读数据，输出Word到指定目录。

---

## 十、审查修复对照

| 审查项 | 严重度 | V2修复 |
|--------|--------|--------|
| reader遗漏reasoning/system_prompt等字段 | 必须改 | 扩展为session_meta+messages全23字段(3.1) |
| parser语义字段职责越界 | 必须改 | 纯确定性，purpose/summary移除(3.2) |
| LLM零降级策略 | 必须改 | 超时+重试+降级路径+中间结果落盘(第五章) |
| classifier/narrator prompt不一致 | 建议改 | 统一为全六角度，narrator拿到完整数据(3.3, 3.4) |
| 大session token超限 | 建议改 | 30K token/50轮阈值，自动分块20轮/组(第六章) |
| narrator输出无格式约束 | 建议改 | JSON schema强约束(3.4) |
| classifier缺few-shot示例 | 建议改 | prompt含2个示例+完整schema(3.3) |
| writer跨项目依赖 | 建议改 | 排版函数拷贝到项目内，源文件注明(3.5) |
| 无断点续跑 | 建议改 | .cache/中间结果+--no-cache强制刷新(5.3, 3.6) |

---

## 十一、成本透视 —— 数据来源与方法论

### 11.1 为什么不需要额外埋点

state.db的messages表和sessions表在Hermes写入时已经记录了完整的token统计：

- `messages.token_count`：每条消息的token数（含reasoning_tokens）
- `messages.finish_reason`：LLM调用是否正常结束（"stop"/"length"/"tool_calls"）
- `sessions.input_tokens / output_tokens / reasoning_tokens`：整个session的汇总
- `sessions.estimated_cost_usd`：DeepSeek API返回的成本估算

晒阳不需要在Agent运行时插桩——成本数据是Hermes已经记录的副作用。reader读到什么，parser就算什么。

### 11.2 成本指标的定义与数据来源

| 指标 | 计算方式 | 数据来源 | 精确度 |
|------|----------|----------|--------|
| 总输入token | 直接读取 | sessions.input_tokens（DeepSeek API返回值） | 精确 |
| 总输出token | 直接读取 | sessions.output_tokens（DeepSeek API返回值） | 精确 |
| 单条消息token | 直接读取 | messages.token_count（Hermes写入的API返回值） | 精确 |
| 成本(USD) | 直接读取 | sessions.estimated_cost_usd（DeepSeek API返回值） | 精确 |
| 有效token | messages.token_count累加筛选 | 数据读取+规则分类 | 精确 |
| 开销token | 总token - 有效token | 精确数据减法 | 精确 |
| 开销占比 | 开销token/总token | 精确数据除法 | 精确 |
| 浪费token | finish_reason!="stop"的消息token之和 | 精确数据+规则筛选 | 近似（无法区分"必要的重试"和"浪费的重试"） |
| 并行节省估算 | 并行轮次总token × 0.3 | 推理计算，非实测 | 估算（对话未以串行方式实际发生过） |

**关键区分：前三类（总token/单条token/成本）是DeepSeek API每次调用后返回的真实数据，Hermes记录到state.db中。晒阳直接读取，不计算、不估算、不推测。**

后两类（浪费/节省）是基于真实数据做的工程判断——数据本身是真的，但"这是不是浪费""省了多少"需要推理。架构文档中用"估算"一词仅指并行节省（因无法实测），浪费token用"近似"——数据精确但分类有主观性。

### 11.3 成本分析不能完全自动化

成本数据计算是确定性的（parser层），但以下几个判断需要LLM（narrator层）：

- "这次审查多花了$0.015，值不值？"——需要LLM判断审查发现的9个问题的严重程度
- "工具调用串行化是否合理？"——需要LLM理解工具之间的依赖关系（pymupdf必须先装才能提取）
- "浪费的2000 token是否可避免？"——需要LLM理解pymupdf venv未安装是环境问题还是设计问题

所以成本透视分两层：数字由parser算（零LLM成本），解读由narrator写（计入narrator的LLM调用，不额外增加调用次数）。

### 11.4 成本数据在报告中的呈现策略

**双层呈现：全流程嵌入 + 全局视角。**

**第一块（全流程展示）——只标异常：**

成本数据不跟着每个步骤走。只在两种情况下嵌入：

- 成本热点：某步token消耗显著高于平均值（>2倍均值）。标注"本步消耗X tokens，是全对话第N贵的步骤，原因：..."
- 成本节省：某决策显著降低了成本（如选patch而非write_file）。标注"本决策节省约X tokens，对比方案Y"

正常消耗不出现。避免每步都塞数字打断叙事流。判定标准：parser输出的by_turn数据中，标记efficiency为"low"（热点）或"high_with_savings"（节省点）的轮次才嵌入。

**第三块（成本透视）——全局视角：**

保留完整数据——per-turn成本表、所有步骤的token消耗、开销分解、趋势分析。读者想深入了解时翻到这一章即可。

**逻辑：**
```
全流程(定性+异常定量) → 六角度(结构分析) → 成本透视(完整定量) → 核心发现(交叉)
```
读者第一遍看流程时不被数字淹没，发现异常时知道"这里贵了/省了"，想深究时翻到第三块看完整数据。
