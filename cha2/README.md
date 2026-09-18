# Agent 第二章学习

## 本章学习目标

第二章主要学习 Agent 的模型服务、工具调用以及本地大模型部署。

这一章我首先完成了实验 2-1：Local LLM Serving。

通过这个实验，我在自己的 Mac 上使用 Ollama 部署了 Qwen3:0.6B，并通过 Python Agent 实现了工具调用、多工具并行调用以及性能测试。

---

# 实验部分

## 实验 2-1：Local LLM Serving

### 实验目标

1. 在本地部署大语言模型
2. 理解 LLM 与 Agent 工具之间如何通信
3. 学习 Tool Calling
4. 学习多工具调用
5. 理解 TTFT、吞吐量和 KV Cache
