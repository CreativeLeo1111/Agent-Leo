# 实验 5-1：DeepSeek 与 Qwen 的跨模型轨迹接管

> 基于 [bojieli/ai-agent-book 第 5 章实验 5-1](https://github.com/bojieli/ai-agent-book/tree/main/chapter5/provider-failover) 的改造与单轮复现。本文记录的是 **2026-09-28** 的本地运行结果，不代表原仓库的正式实验结果。

## 1. 实验目标

验证一个需要多次工具调用的 Agent 在执行到一半时，能否从 DeepSeek 切换到 Qwen（阿里云百炼 DashScope），或反向切换，并继续完成任务。比较三种历史轨迹处理方式：`naive`（直传）、`strip`（剥离思考信息）和 `neutral`（转换为中立表示）。观察切换后的首个请求、任务完成情况、重复工具调用、轮数及 token 消耗。

任务需要取得机票、住宿、餐费和汇率四项数据，并计算人民币总额。完成前两次工具调用后，向当前服务方人为注入连续故障，触发熔断与模型切换；目标模型随后完成剩余步骤。注入的状态码为 `429、429、503`，**并非厂商真实故障**。

## 2. 为什么改造

原实验使用 Moonshot、Anthropic 和 Google Gemini，重点检验不同厂商的思考内容、签名和工具调用结构能否跨接口移交。本次改用可用的 DeepSeek 与 Qwen（DashScope）服务，保留故障注入、熔断、切换和任务验收的核心流程，以便在两家模型之间先完成一个可运行的双向对照。

这改变了实验覆盖面：两家模型均通过 OpenAI 兼容风格的消息与工具调用接口接入，不能用本次结果替代原实验对 Anthropic `signature`、Gemini `thoughtSignature` 等厂商专有字段的测试。

## 3. 环境与实现假设

- 在本地 `provider-failover` 实验目录运行改造后的 `run_handoff.py`，使用相对路径 `../../.venv/bin/python` 指向 Python 虚拟环境。
- 已为 DeepSeek 和 DashScope 分别配置有效的 API 凭据及网络访问。凭据只存放在本地环境变量或未提交的配置文件中；本文不包含密钥。
- 运行输出保存在 `validation/runs/exp5-1-handoff-<UTC 时间戳>/`。本次可见记录含 `summary.json`；未在本文复核原始请求、响应、完整轨迹或代码版本。
- 模型标识由一次展示的 `summary.json` 确认：DeepSeek 为 `deepseek-chat`，Qwen 为 `qwen-plus`。其余运行按同一命令配置解读，若期间修改过配置，应以各自运行目录中的记录为准。

## 4. 关键改动（高层）

1. 将原实验的 Moonshot、Anthropic、Gemini 提供方配置替换为 DeepSeek 与 Qwen（DashScope）的客户端和模型配置。
2. 保留“第二次工具调用后注入故障 → 熔断 → 切换提供方 → 继续执行”的控制流程。
3. 将待测方向设为 `deepseek:qwen` 和 `qwen:deepseek`，每个方向运行 `naive`、`strip`、`neutral` 三种策略。
4. 继续记录首个接管请求的 HTTP 状态、四项数据是否齐备、总额是否正确、切换前已完成工具是否被重复调用，以及切换后的轮数和 token 数。

以上是根据本次运行命令和结果确认的功能层面改动；具体函数、文件差异与依赖版本需要结合实际代码仓库核对。

## 5. 运行命令

以下命令均在改造后的 `provider-failover` 目录执行：

```bash
# 先单独验证反方向的中立策略
../../.venv/bin/python run_handoff.py --pairs qwen:deepseek --arms neutral

# 两个方向 × 三种策略，共 6 个单元
../../.venv/bin/python run_handoff.py

# 完整运行后，再单独重复一次反方向的中立策略
../../.venv/bin/python run_handoff.py --pairs qwen:deepseek --arms neutral
```

在这组命令之前，还完成了一次 `deepseek:qwen` 的 `neutral` 单元。检查结果可使用：

```bash
LATEST=$(ls -dt validation/runs/exp5-1-handoff-* | head -1)
cat "$LATEST/summary.json"
```

## 6. 实测结果

### 6.1 完整的六单元运行

运行目录：`validation/runs/exp5-1-handoff-20260928T134521Z`。

| 切换方向 | 策略 | 首个接管请求 | 数据齐备 | 总额正确 | 重复调用 | 切换后轮数 | 切换后 token |
|---|---|---:|:---:|:---:|---:|---:|---:|
| DeepSeek → Qwen | `naive` | 200 | True | True | 0 | 2 | 215 |
| DeepSeek → Qwen | `strip` | 200 | True | True | 0 | 2 | 215 |
| DeepSeek → Qwen | `neutral` | 200 | True | True | 0 | 2 | 227 |
| Qwen → DeepSeek | `naive` | 200 | True | True | 0 | 2 | 321 |
| Qwen → DeepSeek | `strip` | 200 | True | True | 0 | 2 | 188 |
| Qwen → DeepSeek | `neutral` | 200 | True | True | 0 | 2 | 214 |

该次运行的 **6/6 个单元** 均成功接管、取得完整数据、算对总额，且没有重复调用；全部在切换后两轮完成。

### 6.2 另外三次单独运行

这些是独立运行，不能与上表合并为每个策略的多次重复实验。

| 运行目录 | 切换方向 | 策略 | 首个接管请求 | 数据齐备 | 总额正确 | 重复调用 | 切换后轮数 | 切换后 token |
|---|---|---|---:|:---:|:---:|---:|---:|---:|
| `exp5-1-handoff-20260928T134310Z` | DeepSeek → Qwen | `neutral` | 200 | True | True | 0 | 2 | 215 |
| `exp5-1-handoff-20260928T134512Z` | Qwen → DeepSeek | `neutral` | 200 | True | True | 0 | 2 | 200 |
| `exp5-1-handoff-20260928T134601Z` | Qwen → DeepSeek | `neutral` | 200 | True | True | 0 | 2 | 217 |

第一项的 `summary.json` 还显示 `error: null`、`switch_after_tool_calls: 2`、`outage.injected: true` 和故障注入序列 `[429, 429, 503]`。其他两项的表格数据来自终端输出。

## 7. 结果分析

- **接管可行性：** 在这次测试的 DeepSeek 与 Qwen 配置下，两种方向和三种策略均通过。这个结论只针对已运行的任务、模型版本与接口配置。
- **重复调用：** 六单元运行中均为 0，说明模型没有重新执行切换前已完成的工具。但任务状态主要体现在工具结果里，尚不足以判断思考信息对更复杂任务是否必要。
- **token 差异：** Qwen → DeepSeek 方向的单次完整运行中，`naive` 为 321、`strip` 为 188、`neutral` 为 214；`strip` 比 `naive` 少 133 token，约 41.4%。这只是单次对照，不能据此断言 `strip` 稳定最省。`neutral` 在该方向的三次独立运行分别为 200、214、217，也显示结果会波动。
- **解释边界：** 三种策略都成功，可能与两侧使用相近的 OpenAI 兼容工具调用格式有关；这是基于接口形式和本次现象的推断，不是对底层协议差异的独立测量。

## 8. 与原实验的差异

| 项目 | 原仓库实验 | 本次改造 |
|---|---|---|
| 提供方 | Moonshot、Anthropic、Google Gemini | DeepSeek、Qwen（DashScope） |
| 方向与策略 | 6 个有向厂商组合 × 3 种策略，共 18 个单元 | 2 个方向 × 3 种策略，共 6 个单元 |
| 主要协议难点 | 跨厂商思考签名、工具调用结构等差异 | 两种 OpenAI 兼容接口之间的接管 |
| 报告的首个接管请求成功数 | `naive` 3/6、`strip` 4/6、`neutral` 6/6 | 三种策略各 2/2；合计 6/6 |

原仓库记录的是另一组提供方在其正式运行条件下的结果，不能直接将成功率或 token 数与本次运行做性能排名。原实验的设计和正式结果见[项目 README](https://github.com/bojieli/ai-agent-book/blob/main/chapter5/provider-failover/README.md)；实验目标见[第 5 章正文](https://github.com/bojieli/ai-agent-book/blob/main/book-en/chapter5.md)。

## 9. 局限性

1. 六单元矩阵每格只运行一次；额外单独运行只覆盖 `neutral`，无法估计三种策略各自的成功率、均值或标准差。
2. 故障由测试程序注入，不代表 DeepSeek 或 DashScope 发生真实故障。
3. 此处只列出终端结果及一次 `summary.json` 检查；未审计所有运行的原始 HTTP 响应和工具轨迹。
4. `切换后 token` 按运行输出原样记录；在未核对计量代码前，不将其解释为输入 token、输出 token 或费用。
5. 未记录精确提交、依赖锁定版本、API 日期版本及请求参数；因此其他人重跑可能得到不同的 token 数甚至不同的行为。

## 10. 复现方式

1. 获取[原仓库](https://github.com/bojieli/ai-agent-book)，进入 `chapter5/provider-failover`，应用与本次相同的 DeepSeek / Qwen 适配代码。**仅凭本文不能重建这部分改造**；公开复现时应同时提交代码差异或明确的提交链接。
2. 按所用代码版本的依赖说明建立 Python 虚拟环境，并在本地配置 DeepSeek 与 DashScope 的 API 凭据。不要把 `.env`、密钥或包含授权头的原始请求提交到 GitHub。
3. 在实验目录执行第 5 节的完整运行命令。若虚拟环境位置不同，将 `../../.venv/bin/python` 换成实际 Python 路径。
4. 保存每次运行的 `summary.json`，逐项检查 HTTP 状态、数据完整性、总额正确性、重复调用、轮数和 token 数。若要验证厂商真实报错，还需检查脱敏后的原始响应。
5. 正式比较策略时，对每个方向与策略等次数重复运行，固定代码提交、模型标识和请求参数，再报告每格的成功次数、token 分布与失败原因。

### 本次可核对的运行目录

```text
validation/runs/exp5-1-handoff-20260928T134310Z
validation/runs/exp5-1-handoff-20260928T134512Z
validation/runs/exp5-1-handoff-20260928T134521Z
validation/runs/exp5-1-handoff-20260928T134601Z
```


