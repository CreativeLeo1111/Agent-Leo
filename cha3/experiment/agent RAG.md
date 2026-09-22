# Experiment 3-8：Agentic RAG vs Non-Agentic RAG

## 1. 实验目的

本实验来自《深入理解 AI Agent》第 3 章 Agentic RAG 实验，主要比较两种检索增强生成方式：

- **Non-Agentic RAG**：对原始问题进行一次检索，然后直接生成答案。
- **Agentic RAG**：使用 ReAct 思路，根据当前证据决定是否继续检索、改写查询或补充证据。

本实验重点观察：

1. Agentic RAG 是否能够自主拆解复杂问题；
2. Agent 是否能够生成新的检索 Query；
3. 多轮检索是否能够补充缺失证据；
4. Agentic RAG 与一次性 RAG 在复杂问题上的差异。

原项目：

https://github.com/bojieli/ai-agent-book/tree/main/chapter3/agentic-rag

---

## 2. 实验环境

### 设备

- MacBook Air M1
- macOS

### Python 与工具

```text
Python 3.12.14
uv 0.12.13
Git 2.50.1
```

### LLM

```text
Provider: DashScope
Model: qwen3.7-plus
```

### Knowledge Base

```text
KB_TYPE=offline
Retriever=BM25
```

### 离线法律知识库规模

```text
288 篇文档
21372 个法条 Chunk
```

---

## 3. 环境安装

进入项目根目录：

```bash
cd /Users/luwangqiang/Desktop/ai-agent-book
```

安装 Chapter 3 依赖并创建 Python 3.12 环境：

```bash
uv sync --locked --python 3.12 --extra ch3
```

激活虚拟环境：

```bash
source .venv/bin/activate
```

进入实验目录：

```bash
cd chapter3/agentic-rag
```

配置 `.env`：

```text
DASHSCOPE_API_KEY=自己的API_KEY
KB_TYPE=offline
LLM_PROVIDER=dashscope
LLM_MODEL=qwen3.7-plus
```


---

## 4. 实验一：Offline Retrieval Compare

首先运行项目提供的离线对比实验：

```bash
python compare_offline.py
```

实验结果如下：

| 问题类型 | Single Retrieval | Decomposed Retrieval |
|---|---:|---:|
| 全部问题 | 48% | 100% |
| 简单问题 | 100% | 100% |
| 复杂问题 | 8% | 100% |

平均检索次数：

```text
1.0 → 1.3
```

### 结果理解

简单问题中，一次检索通常已经能够找到关键证据，因此两种方法差别不大。

复杂问题中：

```text
Single Retrieval:      8%
Decomposed Retrieval: 100%
```

说明当一个问题包含多个证据需求时，将问题分解后分别检索，可以显著提高 Evidence Recall。

需要注意：`compare_offline.py` 中的分解查询是预先标注的 subqueries，主要用于单独验证“检索策略”的作用，还不是完整的实时 ReAct Agent。

---

## 5. 测试问题

本实验选择以下复杂问题：

```text
醉酒过失致人重伤且有盗窃前科如何量刑
```

该问题实际上包含多个证据需求：

```text
过失致人重伤如何量刑？
        +
醉酒是否影响刑事责任？
        +
盗窃前科是否导致从重处罚？
```

这类问题非常适合观察 Agent 是否会主动拆分问题并进行多步检索。

---

## 6. Non-Agentic RAG

运行：

```bash
python main.py \
  --kb-type offline \
  --provider dashscope \
  --model qwen3.7-plus \
  --query "醉酒过失致人重伤且有盗窃前科如何量刑" \
  --mode non-agentic \
  --verbose
```

日志中可以看到：

```text
Offline BM25 search returned 5 results
```

Non-Agentic RAG 的基本流程为：

```text
User Query
    ↓
BM25 Retrieval
    ↓
Top-K Documents
    ↓
LLM
    ↓
Answer
```

这里只有一次 Retrieval。

本次实验中，模型最终回答：

```text
根据提供的上下文，没有包含关于
“醉酒过失致人重伤且有盗窃前科如何量刑”
的相关信息。
```

一次检索返回的内容主要涉及：

```text
故意伤害罪
虐待部属罪
危险方法致人重伤
```

但没有召回回答问题真正需要的几个关键法条，例如：

```text
刑法第十八条
刑法第六十五条
刑法第二百三十五条
```

这说明：当原始 Query 同时包含多个概念时，一次词法检索可能无法把所有必要证据都召回。

---

## 7. Agentic RAG

运行：

```bash
python main.py \
  --kb-type offline \
  --provider dashscope \
  --model qwen3.7-plus \
  --query "醉酒过失致人重伤且有盗窃前科如何量刑" \
  --mode agentic \
  --verbose
```

Agentic RAG 使用类似 ReAct 的循环：

```text
Reason
   ↓
Search
   ↓
Observe
   ↓
Reason
   ↓
Search Again
   ↓
...
   ↓
Final Answer
```

与 Non-Agentic 不同，Agent 可以根据当前证据决定是否继续搜索。

---

## 8. 保存并分析 Agent 检索轨迹

为了观察 Agent 实际生成了哪些搜索词，保存日志：

```bash
python main.py \
  --kb-type offline \
  --provider dashscope \
  --model qwen3.7-plus \
  --query "醉酒过失致人重伤且有盗窃前科如何量刑" \
  --mode agentic \
  --verbose 2>&1 | tee agentic_trace.log
```

提取关键轨迹：

```bash
grep -E 'ITERATION|TOOL CALL|Query:' agentic_trace.log
```

本次真实运行中，Agent 进行了多轮检索。

### Iteration 1：先拆分问题

```text
过失致人重伤罪 量刑标准
醉酒 过失犯罪 刑事责任
前科 盗窃 量刑 从重处罚
```

可以看到，Agent 没有直接重复原始问题，而是把复杂问题拆成了三个证据方向：

1. 过失致人重伤的量刑；
2. 醉酒与刑事责任；
3. 前科是否会导致从重处罚。

### Iteration 2：从概念检索转向精确法条检索

```text
过失致人重伤罪 第二百三十五条
醉酒的人犯罪 刑事责任 第十八条
累犯 前科 量刑情节
```

这一轮已经开始针对具体法条进行搜索。

可以理解为：

```text
概念检索
   ↓
发现可能对应的法律条文
   ↓
精确法条检索
```

### Iteration 3：进一步判断“前科”和“累犯”

```text
累犯 第六十五条 第六十六条
量刑原则 第六十一条 犯罪事实 情节
```

这里体现了一个重要的多跳过程：

```text
有盗窃前科
   ↓
是否构成累犯？
   ↓
累犯成立条件是什么？
   ↓
本次属于过失犯罪
   ↓
是否属于累犯规定中的例外？
```

Agent 并没有简单地把“有前科”等同于“必须从重处罚”。

### Iteration 4：补充验证

```text
第六十五条 累犯 刑罚执行完毕 五年内 再犯
第六十一条 量刑原则 犯罪事实 性质 情节 社会危害程度
```

这一轮主要是在补充和验证前面的判断。

### Iteration 5：停止检索并回答

进入：

```text
ITERATION 5/10
```

之后没有新的 Tool Call。

这说明 Agent 判断当前证据已经足够，因此停止检索并生成最终答案。

本次轨迹可以总结为：

```text
用户复杂问题
       ↓
Iteration 1：拆分多个证据需求
       ↓
Iteration 2：寻找具体法条
       ↓
Iteration 3：补充累犯/前科判断
       ↓
Iteration 4：验证量刑条件
       ↓
Iteration 5：证据足够，停止检索
       ↓
Final Answer
```

---

## 9. Agentic RAG 找到的关键证据

### 《刑法》第二百三十五条

```text
过失伤害他人致人重伤
→ 三年以下有期徒刑或者拘役
```

### 《刑法》第十八条

```text
醉酒的人犯罪
→ 应当负刑事责任
```

### 《刑法》第六十五条

```text
累犯应当从重处罚

但过失犯罪属于累犯规定中的例外情形之一
```

Agent 最终可以把不同检索步骤获得的证据组合起来，再生成答案。

---

## 10. Agentic vs Non-Agentic 对比

| 对比项目 | Non-Agentic RAG | Agentic RAG |
|---|---|---|
| Query | 原始问题 | 自动生成多个 Query |
| 检索方式 | 一次检索 | 多轮检索 |
| 问题拆解 | ❌ | ✅ |
| 根据结果继续搜索 | ❌ | ✅ |
| 过失致人重伤证据 | ❌ | ✅ |
| 醉酒刑责证据 | ❌ | ✅ |
| 累犯相关证据 | ❌ | ✅ |
| 最终结果 | 无法基于当前证据回答 | 可以基于检索证据回答 |

---

## 11. 为什么 Agentic RAG 更适合复杂问题？

传统 RAG 更接近：

```text
Query
 ↓
Retrieve
 ↓
Generate
```

如果第一次 Retrieval 没有召回关键证据，后面的 LLM 即使能力很强，也缺少足够上下文。

Agentic RAG 则更接近：

```text
Query
 ↓
Reason
 ↓
Retrieve
 ↓
Observe
 ↓
发现证据缺口
 ↓
Rewrite / Decompose Query
 ↓
Retrieve Again
 ↓
Answer
```

它的关键不是单纯“多搜索几次”，而是：

> 根据已经找到的证据，判断还缺少什么，然后主动改变下一步检索行为。

---

## 12. 本次实验的核心理解

通过这次实验，我对 Agentic RAG 的理解是：

### 普通 RAG

更接近一个静态的信息获取过程：

```text
一个问题
→ 一次搜索
→ 一次生成
```

### Agentic RAG

更接近一个动态的信息获取过程：

```text
问题理解
→ 搜索
→ 观察结果
→ 判断证据缺口
→ 生成新查询
→ 再搜索
→ 停止条件
→ 最终回答
```

因此对于以下问题，Agentic RAG 可能更有优势：

- 多跳问题；
- 复杂组合问题；
- 模糊问题；
- 跨文档问题；
- 需要逐步补证据的问题。

---

## 13. Agentic RAG 的代价与注意事项

Agentic RAG 并不是没有代价。

相比一次性 RAG，它通常会带来：

```text
更多 LLM 调用
更多 Retrieval 调用
更高 Token 消耗
更高延迟
更高运行成本
```

另外，Agent 的工具调用路径由 LLM 动态生成，因此不同运行之间可能出现不同的：

```text
Query
Iteration 数
Retrieval 次数
```

本实验中，同一个问题在不同运行中就出现过不同长度的检索轨迹。

因此如果要严谨评估 Agentic RAG，不能只依赖单次运行，而应该进行多次重复实验，并统计：

- Evidence Recall；
- Answer Accuracy；
- Retrieval 次数；
- Token Usage；
- Latency；
- Cost。

---

## 14. 下一步计划

后续计划继续学习和实验：

- Agentic RAG 的停止条件；
- Query Rewrite；
- Query Decomposition；
- Retrieval Evaluation；
- Reranker；
- Hybrid Search；
- Vector Retrieval；
- BM25 与向量检索的对比；
- Agentic RAG 的 Accuracy / Token / Latency trade-off；
- 在自己的知识库上实现 Agentic RAG。

---

## 15. 本次实验总结

本实验最重要的收获不是“成功运行了 Agentic RAG”，而是实际观察到了完整的检索轨迹：

```text
复杂问题
   ↓
拆分证据需求
   ↓
生成多个检索 Query
   ↓
根据返回结果继续补充证据
   ↓
逐渐转向更精确的法条搜索
   ↓
判断证据是否已经足够
   ↓
停止检索并生成答案
```

这说明 Agentic RAG 的核心价值在于：

> **把一次性检索变成一个由模型控制的、可迭代的证据获取过程。**

