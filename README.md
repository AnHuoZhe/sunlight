# Sunlight

> 把 Agent 的 ReAct 循环晒在太阳底下——读对话记录，拆成九步流程，标注每一步对应什么，生成本地 Word 报告。

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![Python 3.11+](https://img.shields.io/badge/python-3.11+-blue.svg)](https://www.python.org/)

## Why This Exists

Agent 的 ReAct 循环是个黑盒——你看得见输入输出，看不见中间的意图解析、任务拆解、工具选择。出了问题不知道哪步判断错了。

Sunlight 把一次完整的 Agent 对话摊开：用户输入 → 静态前缀 → 意图解析 → 任务拆解 → 工具选择 → 轨迹累积 → 约束检查 → 生成回答 → 验证纠正。每一步标注对应什么，生成 Word 报告。

## Quick Start

```bash
git clone https://github.com/AnHuoZhe/sunlight.git
cd sunlight
pip install pyyaml python-docx
```

```python
# 读 Hermes 对话数据库，生成分析报告
python main.py
```

输出：Word 报告保存在 `samples/` 目录。

## 为什么不是 Agent 而是工作流

Sunlight 是纯工作流——5 段管道（读→解析→分类→叙述→输出），2 次 LLM 调用，不自主执行任何操作。它分析 Agent 但不做 Agent。

设计原因：Agent 分析自己的推理时不是在"回忆"（LLM 没有记忆），而是在"构造"听起来合理的推理链条。这种自我美化是结构性的——Agent 无法自主发现自己在美化。Sunlight 站在外部看，不受这个偏差影响。

## 五段管道

| 阶段 | 功能 |
|------|------|
| Reader | 读 Hermes state.db 对话记录 |
| Parser | 解析消息流转和工具调用 |
| Classifier | 六角度分类（LLM/上下文/工具/约束/验证/纠正） |
| Narrator | 生成自然语言叙述 |
| Writer | 输出 Word 报告 |

## License

MIT © [AnHuoZhe](https://github.com/AnHuoZhe)
