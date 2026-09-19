# 实验 2-1：本地 LLM 服务部署与工具调用

## 1. 实验目的

本实验主要是第一次完整体验一个本地 Agent 的运行过程。

我希望通过这个实验理解：

1. 如何在本地运行一个大语言模型
2. LLM 如何决定调用工具
3. Agent Harness 如何真正执行工具
4. Tool Result 如何重新返回给模型
5. 多个工具如何并行调用
6. KV Cache 对模型响应速度的影响

这个实验我理解了
用户提出问题
    ↓
LLM 分析问题
    ↓
LLM 判断是否需要工具
    ↓
生成 Tool Call
    ↓
Agent Harness 执行工具
    ↓
返回 Tool Result
    ↓
重新发送给 LLM
    ↓
LLM 生成最终回答


本地模型：

```text
Qwen3:0.6B
