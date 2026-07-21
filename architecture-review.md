# 晒阳架构文档 · 审查报告

> 审查类型：架构审查
> 方案规模：小方案（用户指定）
> 审查项：数据流 / 失败模式 / 上下文工程(prompt设计) / 工具调用(I/O契约) / 可靠性(LLM降级)
> 审查日期：2026-07-21

---

## 审查前验证

在开始审查前，实际查询了 Hermes state.db 的真实 schema（两份表）：

- **messages 表**：23个字段。架构文档只提到 role/content/timestamp/tool_calls，但实际包含 `reasoning`（LLM推理链）、`reasoning_content`、`tool_call_id`（工具结果关联）、`tool_name`、`finish_reason`、`token_count` 等关键分析字段。
- **sessions 表**：38个字段。包含 `system_prompt`（约束角度的核心数据源）、`model`、`input_tokens/output_tokens/reasoning_tokens`、`estimated_cost_usd`、`parent_session_id`、`message_count`、`tool_call_count` 等。

架构文档描述的 reader 只提取了约1/5的可用字段——这不是简化设计，是信息断层。

---

## 必须改

### 1. reader 模块遗漏关键数据源，导致六角度分析的证据链断裂

**问题**：架构定义的 reader 输出只含 `{session_id, messages[{role, content, timestamp, tool_calls}]}`。但 actual state.db 中以下字段对六角度分析至关重要：

| 字段 | 所属表 | 对哪个角度至关重要 | 当前架构 |
|------|--------|-------------------|----------|
| `reasoning` / `reasoning_content` | messages | LLM——直接记录内部思考链 | 未读取 |
| `tool_call_id` / `tool_name` | messages | 工具——关联调用与结果 | 未读取 |
| `finish_reason` | messages | 可靠性——LLM调用是否被截断 | 未读取 |
| `system_prompt` | sessions | 约束——规则来源 | 未读取 |
| `model` | sessions | LLM——模型能力基线 | 未读取 |
| `token_count` / `input_tokens` / `output_tokens` | messages/sessions | 成本/规模量化 | 未读取 |

**为什么必须改**：样本报告中的高质量分析（如"调用频率X次，决策类型分布"、"活跃约束列表"、"并行vs串行调用模式"）依赖的就是这些字段。不读它们，LLM只能从文本推测，准确率远低于结构化数据直接提供。

**修复方向**：reader 输出扩展为 `{session_meta: {...}, messages: [{..., reasoning, tool_call_id, tool_name, finish_reason, token_count}]}`。session 的 system_prompt、model、token 统计直接从 sessions 表读取。

---

### 2. parser 声称"确定性解析"但输出了需要语义理解的字段

**问题**：parser 输出中的 `assistant_actions` 包含：
- `{"type": "llm_call", "purpose": "加载技能"}` — `purpose` 是语义判断，确定性解析做不到
- `{"type": "tool_call", ..., "result_summary": "..."}` — 总结工具结果需要理解返回值

架构说 parser 是纯规则层，但这些字段的来源没有定义。

**为什么必须改**：模块职责边界模糊——parser 做了 classifier 该做的事（语义理解）。实现阶段必然发现 parser 需要调 LLM 或引入不可靠的启发式规则。两种后果：(a) parser 变成事实上的 LLM 调用者，LLM 调用次数从"2次"变成 3+ 次；(b) parser 输出这些字段为空或错误，下游崩溃。

**修复方向**：二选一——
- 方案A：parser 只输出确定性数据（轮次边界、工具调用名、时间戳），`purpose` 和 `result_summary` 移到 classifier 或 narrator 的 LLM prompt 中生成。
- 方案B：承认 parser 需要 LLM，将 classifier 的语义层并入 parser，调整模块边界和 LLM 调用次数为 3 次。

---

### 3. LLM 调用零降级策略——任一次失败 = 全管线阻塞

**问题**：架构定义 2 次 LLM 调用（classifier 语义层 + narrator），没有提到任何失败处理：
- 无重试逻辑
- 无超时设置
- 无 fallback 输出（如只用规则层分类结果生成简化报告）
- 无中间结果持久化——narrator 失败后 classifier 结果丢失

**为什么必须改**：DeepSeek API 不可用时，用户跑完 reader→parser→classifier规则层 后收到一个崩溃或空白。用户不知道是 API 挂了、超时了、还是 prompt 太长被拒绝。这个工具的设计初衷是"确定性工作流"，但 LLM 调用是全流程的单点故障源。

**修复方向**：
1. classifier 的规则层结果必须落盘（JSON），不依赖 LLM 成功。
2. 如果 LLM 层失败，narrator 回退到只用规则层结果 + 原始对话生成简化报告，并在报告中标注"本次分析未包含LLM语义层分类"。
3. LLM 调用加超时（60s）和重试（3次，指数退避）。

---

## 建议改

### 4. classifier 和 narrator 的 prompt 定义不一致

**问题**：classifier 的 LLM prompt 只要求识别三类（LLM决策意图/验证行为/纠正行为），而 narrator prompt 要求从六个角度全部展开分析。中间的"上下文/工具/约束"三个角度只有规则层覆盖，语义层完全没有 LLM 输入。narrator 被要求凭空分析这三个角度。

**为什么建议改**：narrator 拿到的上下文在三个角度上只有规则层的浅层标签（"调了X工具"），而样本报告中的高质量分析（如"工具选择逻辑：需要用户背景→skill_view"、"约束执行效果：在第8轮需求审查中约束⑤生效"）需要深层语义理解。让 narrator 在没有足够输入的情况下硬分析，输出质量必然下降。

**修复建议**：要么把 classifier 的 LLM prompt 扩展到全部六个角度，要么在 narrator 的 prompt 中给足原始数据（对话原文 + 规则分类 + 工具调用记录），让 narrator 自己补全语义缺口。推荐后者——减少一次 LLM 调用的同时保证 narrator 有完整上下文。

---

### 5. 大 session（50+轮）的 token 超限风险未处理

**问题**：架构说"不是每轮调一次——把所有轮次打包成一次 prompt"。对于 10 轮对话这可行，但如果用户分析一个 80 轮的长 session，单次 prompt 会远超 DeepSeek 的上下文窗口。架构没有提到截断、分块、或摘要策略。

**为什么建议改**：工具的用户就是 Hermes 用户自己，而 Hermes 的一个 session 完全可以有 50+ 轮。不处理的后果是到运行时才发现 token 超限，然后报错退出。

**修复建议**：
- 在 main.py 启动时估算 token 数（消息数 × 平均长度 ÷ 4 ≈ token 数），超过阈值时自动切换为分块模式。
- 分块策略：每 20 轮一组，每组独立做 classifier，最后 narrator 拿到所有组的分类摘要合成整体报告。

---

### 6. narrator prompt 无输出格式约束，代码解析自由文本不可靠

**问题**：架构定义了 narrator 输出结构（overview / turn_by_turn / angle_summary / findings / improvements），但 narrator 的 LLM prompt 没有要求输出 JSON 或任何结构化格式。这意味着 narrator.py 必须解析自由文本并映射到结构化字段。

**为什么建议改**：自由文本解析极不可靠——LLM 可能把 "findings" 写成 "核心发现"，"improvements" 写成 "改进建议"，或者把所有内容揉在一起不分段。样本报告的质量建立在人工撰写之上，不能假设 LLM 能自然输出相同结构。

**修复建议**：narrator 的 LLM prompt 改为：
```
按以下 JSON 结构输出（只输出 JSON，不要解释）：
{
  "overview": "...",
  "turn_by_turn": [{"turn": N, "title": "...", "narrative": "...", "angles": {"LLM": "...", ...}}],
  "angle_summary": {"LLM": "...", "上下文": "...", "工具": "...", "约束": "...", "验证": "...", "纠正": "..."},
  "findings": ["...", "..."],
  "improvements": ["...", "..."]
}
```
并提供 1 个完整示例（取自样本报告）。

---

### 7. classifier 的 LLM prompt 缺少 few-shot 示例和输出 schema

**问题**：classifier 的 prompt 只有 5 行，没有示例，没有明确的 JSON 输出格式。LLM 需要猜测：(1) "LLM决策意图" 长什么样？(2) "验证行为" 和 "纠正行为" 怎么区分？(3) 输出 JSON 的 key 是什么？

**为什么建议改**：同样的 prompt 给不同模型或同一模型的不同调用，输出格式可能完全不同。没有 schema 约束，classifier.py 无法可靠解析结果。而分类质量直接影响 narrator 的输入质量——上游偏了，下游全偏。

**修复建议**：在 prompt 中加入 2-3 个 few-shot 示例和明确的输出 schema：
```json
{
  "turns": [
    {
      "turn": 1,
      "llm_decision": "概念解释任务，判断为高确定性→直接回答",
      "verification": "自检回答是否覆盖了全部子问题",
      "correction": null
    }
  ]
}
```

---

### 8. writer 对 project-analyzer 的依赖未定义契约

**问题**：架构说 writer "复用 project-analyzer 的 `write_word_report()` 排版函数"。但没有定义：(1) 是 import 还是拷贝代码？(2) 那个函数的输入签名是什么？(3) 如果 project-analyzer 的代码变了怎么办？

**为什么建议改**：跨项目代码依赖没有契约是最常见的"开发时跑通、三个月后崩"的坑。

**修复建议**：如果 project-analyzer 是稳定库，明确 import 路径和版本要求。如果只是一次性复用，直接把函数拷贝到晒阳项目内并注明来源，去掉外部依赖。

---

### 9. 缺少中间结果持久化，无断点续跑能力

**问题**：reader→parser→classifier→narrator→writer 全部在内存中传递，任何一步失败后所有已完成的步骤结果丢失。

**为什么建议改**：这不是性能优化，是开发体验问题。当你反复调试 narrator 的 prompt 时，每次都要重新跑 reader+parser+classifier，浪费时间且消耗 LLM token（classifier 的 LLM 调用）。

**修复建议**：每步完成后将中间结果写入 `output_dir/.cache/{step_name}.json`。main.py 启动时检查缓存，跳过已完成的步骤（除非用户传 `--no-cache`）。

---

## 可忽略

- reader 用 sqlite3 标准库直读 state.db —— 合理，state.db 就是 SQLite。
- .env 配置文件方式 —— 简单直接，够用。
- main.py 纯顺序编排，无循环无分支 —— 符合"确定性工作流"定位。
- 模块数量（5个）和职责划分（读→解析→分类→叙述→输出） —— 管道结构清晰。
- 输出 Word 的字体和排版规格 —— 跟样本报告一致，模板化实现可行。

---

## 整体判断

架构的**管道结构本身正确**——读→解析→分类→叙述→输出的五段式设计是干净的。核心问题不在结构，在三个层面：

**数据层**：reader 对 state.db schema 的理解严重不完整。23 个字段只用了 4 个，实际可用数据是架构描述的 5 倍。`reasoning`（LLM 推理链）和 `system_prompt`（约束来源）这两个对六角度分析至关紧要的字段被完全忽略。这是最大的信息断层——不是"功能不完善"，是"分析的基础数据就没拿全"。

**契约层**：模块间数据契约有模糊和越界。parser 说自己是确定性解析但输出了语义字段；classifier 和 narrator 的 prompt 定义不一致；narrator 的输出格式在代码和 prompt 中不对齐。这些问题不会让代码跑不起来，但会让输出质量不可控——LLM 凑巧输出对了一次不代表下次也对。

**可靠性层**：两次 LLM 调用无任何降级策略。这不只是"加个重试"的问题——如果用户跑了一个 80 轮 session，花了 30 秒解析完，然后 DeepSeek API 超时，整个管线死掉，什么都没留下。对于一个标榜"确定性工作流"的工具，这个单点故障的位置太核心了。

修正优先级：必须改 #1（数据层）> 必须改 #3（可靠性层）> 必须改 #2（契约层）> 建议改各项按需推进。
