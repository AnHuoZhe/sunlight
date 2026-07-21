# 晒阳 (Sunlight)

Agent ReAct 循环透明化分析工具。读取 Hermes 对话 session，按六角度拆解 ReAct 循环，生成 Word 分析报告。

## 一句话

让 Agent 的黑箱在阳光下晒一晒——从用户输入到模型输出的每一步，标注在李博杰《深入理解 AI Agent》书中的对应概念，再从六个角度做系统性分析。

## 六角度分析框架

`Agent = LLM + [上下文 + 工具 + 约束 + 验证 + 纠正]`

- **LLM**：什么时候调用了 LLM、内部思考了什么、做了什么决策、什么时候走了规则
- **上下文**：静态前缀（系统提示词+技能+记忆）和轨迹（对话历史）分别注入了什么
- **工具**：调了什么工具、为什么选它、结果怎么用、失败怎么处理
- **约束**：什么规则在限制 Agent 的行为、每条约束是否生效
- **验证**：Agent 怎么检查自己做得对不对
- **纠正**：错了怎么补救、用户纠正后怎么调整

## 报告结构

两段式：

1. **全流程展示**：从用户输入→加载静态前缀→意图解析→任务拆解→工具选择与执行→轨迹累积→约束检查→生成回答→验证与纠正，每步标注对应李博杰《深入理解 AI Agent》的章节
2. **六角度分析**：按上述六个角度逐一分析本次对话的 ReAct 循环特征

## 快速开始

```bash
# 1. 安装依赖
pip install python-dotenv python-docx

# 2. 配置
cp .env.example .env
# 编辑 .env，填入 DeepSeek API Key 和 state.db 路径

# 3. 运行
python main.py              # 分析最近一次 session
python main.py <session_id>  # 分析指定 session
```

## 配置

| 环境变量 | 说明 | 默认值 |
|----------|------|--------|
| `STATE_DB_PATH` | Hermes state.db 路径 | 无，必填 |
| `DEEPSEEK_API_KEY` | DeepSeek API 密钥 | 无，必填 |
| `OUTPUT_DIR` | 报告输出目录 | `./output` |

## 设计

纯工作流工具，非自主 Agent。五段式管道：读→解析→分类→叙述→输出。LLM 仅在语义分类和叙述生成两处介入（最多 2 次调用），失败有降级策略和中间结果缓存。

详见 [architecture.md](architecture.md)。

## 许可证

MIT
