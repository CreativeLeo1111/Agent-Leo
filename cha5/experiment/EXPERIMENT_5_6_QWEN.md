# Experiment 5-6：Qwen Paper → PPT 对照实验记录

> 将论文 *Attention Is All You Need* 转成 Slidev 演示文稿，比较双 Agent（Proposer–Reviewer）与单 Agent 自审。本文记录的是 `validation/runs/qwen-both-r2/` 的一次正式运行；模型输出有随机性，复跑分数可能变化。

## 实验目标与设计

- **方案 A：双 Agent。** Proposer 阅读论文并生成、修改 `slides.md`，只接收 Reviewer 的结构化文字反馈；Reviewer 每轮只查看最新渲染的 PNG，独立给出视觉问题与评分。
- **方案 B：单 Agent 自审。** 同一个 Agent 生成文稿、查看渲染截图、提出修改，历史图片继续留在同一上下文。
- 用同一套独立 Vision 评委评价两个方案最终版，比较质量、单次请求的上下文峰值和总 Token 消耗。`peak_context_prompt_tokens` 是最大单次 prompt 长度；`total_tokens` 是整次运行的累计用量，两者不能混为一谈。

输入是固定的真实论文 PDF，保留论文原始 Figure 1、Figure 3、Figure 4。生成物是 Slidev 的 Markdown 与逐页 PNG，不是 `.pptx`。

## 项目位置与环境

本机项目目录：

```text
/Users/luwangqiang/Desktop/ai-agent-book/chapter5/paper-to-ppt
```

从仓库根目录初始化 Python 环境（项目 README 推荐的 `uv` 路径）：

```bash
cd /Users/luwangqiang/Desktop/ai-agent-book
uv sync --locked --python 3.12 --extra ch5
source .venv/bin/activate
cd chapter5/paper-to-ppt
npm install
```

如果 Chromium 浏览器组件缺失：

```bash
npx playwright install chromium
```

本项目的 `requirements.txt` 也提供 `pip` 安装方式；使用已有 Python 虚拟环境时可执行 `python -m pip install -r requirements.txt`。需要 Node.js、Slidev 和 Playwright 才能渲染页面。网页界面的 Gradio 依赖也列在 `requirements.txt` 中。

## Qwen API 配置

项目通过 OpenAI 兼容接口调用阿里云百炼。先在项目目录创建 `.env`，填入自己的 Key：

```dotenv
OPENAI_API_KEY=你的千问API_Key
OPENAI_BASE_URL=https://dashscope.aliyuncs.com/compatible-mode/v1
TEXT_MODEL=qwen-plus
VISION_MODEL=qwen3-vl-plus
```

`qwen-plus` 负责生成文字和 Slidev 源码；`qwen3-vl-plus` 负责看渲染图片、审查和最终评分。上面的地址对应中国北京区域，其他区域需使用对应的兼容接口地址。`.env` 已在 `.gitignore` 中，**不要把真实 API Key 上传 GitHub**。正式运行的模型信息也写在 `comparison_summary.json` 中。

## 运行前检查

```bash
python demo.py --smoke
python demo.py --dry-run
```

`--smoke` 只检查 Slidev 渲染链路；`--dry-run` 用脚本化文稿和启发式 Reviewer 走通离线循环。两者不调用 Qwen，也不能代表真实 PPT 质量。完成后再做一轮真实 API 小规模检查：

```bash
python demo.py --mode dual --max-rounds 1
python demo.py --mode single --max-rounds 1
```

## 排错与修复记录

### 双 Agent：Proposer 未通过源码密度约束

初次真实运行报 `Proposer failed the source density contract after 3 attempts`。曾出现普通页面超过 **每页最多 4 条 bullet**，以及原论文图页面夹杂正文、额外说明等情况。重新运行后仍有页面违规，说明 `qwen-plus` 对这类严格版式规则的遵循不够稳定。

在 `agents.py` 的 Proposer 提示词中明确写出 18–20 页、每页最多 4 条 bullet、三张原论文图各自独占一页且只含标题、图片和最多一行图注，并要求生成前逐页自检。随后将 Proposer 生成请求的 `temperature` 从 `0.3` 调到 `0.0`。源码验收标准保持不变。双 Agent 单轮真实测试通过，独立评委当时给出 94 分、`pass=True`。

### 单 Agent：原论文图页面格式违规

单 Agent 试跑报 `Single Agent failed the source density contract after 3 attempts`，涉及第 5、16、17 页：原论文图页面多出了正文或额外图注。修复是在 `SelfReviewAgent` 首轮提示词中给出三张图的精确页面模板，指定图片文件、标题和一行图注，并禁止 bullet、正文和第二行图注；同时将该生成请求的 `temperature` 从 `0.3` 调到 `0.0`。同样没有放宽源码验收规则。之后单 Agent 跑通，才执行正式 `both` 对照。

上述修改已经在本机 `agents.py` 中；下次使用当前项目时无需重复编辑。若重新下载或覆盖仓库，应先确认这些改动仍在。

## 正式对照实验

在 `paper-to-ppt` 目录、虚拟环境已激活且 `.env` 已配置的情况下运行：

```bash
python demo.py --mode both --max-rounds 2 \
  --out-dir validation/runs/qwen-both-r2
```

保留旧结果时请换新目录名，例如 `qwen-both-r3`。正式运行会调用付费 API，耗时和用量取决于模型响应、页面数与重试情况。

## 本次结果

结果来源：`validation/runs/qwen-both-r2/comparison_summary.json`。记录显示 `campaign_complete=true`、`official_complete=true`，且正式验收门槛均通过。

| 指标 | 双 Agent | 单 Agent |
|---|---:|---:|
| 最终独立评委评分 | **92** | **94** |
| 最终视觉验收 | `pass=True` | `pass=True` |
| 上下文峰值 `peak_context_prompt_tokens` | **18,555** | **31,837** |
| 总 Token `total_tokens` | **119,161** | **61,914** |

单 Agent / 双 Agent 上下文峰值 = `31,837 / 18,555 ≈ 1.72×`。双 Agent 的 Reviewer 轮次评分为 **94 → 92**。最终独立评委的双 Agent 分数也是 92；两种评分记录应按各自字段解读。

本次两种方案都通过视觉验收，单 Agent 最终分数还高 2 分。因此数据不支持“多 Agent 一定生成更好的 PPT”。双 Agent 的明确优势是**单次上下文峰值更低**：Proposer 不累积图片，Reviewer 每轮从最新截图开始。与此同时，双 Agent 的**累计 Token 更多**，本次不能声称它更省总成本。Reviewer 分数从 94 降到 92 也表明，继续迭代不保证质量单调上升；后续可考虑以验收通过、严重问题数或分数变化作为停止依据。这些结论仅针对本次运行，更多轮次与论文还需另行验证。

## 产物与查看方式

```text
validation/runs/qwen-both-r2/
├── comparison_summary.json       # 正式对照、验收与用量
├── dual_round1_slides.md
├── dual_round1_review.json
├── dual_round2_slides.md          # 双 Agent 最终版
├── dual_round2_review.json
├── single_round1_slides.md
├── single_round2_slides.md        # 单 Agent 最终版
├── source/                         # 论文 PDF、提取文本和原论文图
└── rendered/
    ├── dual_round2/               # 双 Agent 最终逐页 PNG
    └── single_round2/             # 单 Agent 最终逐页 PNG
```

此外，渲染过程也可能在 `slidev_workspace/exports/` 留下 PNG。查看结果与最终图片：

```bash
python -m json.tool validation/runs/qwen-both-r2/comparison_summary.json
open validation/runs/qwen-both-r2/rendered/dual_round2
open validation/runs/qwen-both-r2/rendered/single_round2
```

若想像放映 PPT 一样在浏览器翻页，先从项目目录复制最终版，再启动 Slidev：

```bash
cp validation/runs/qwen-both-r2/dual_round2_slides.md slidev_workspace/slides.md
cd slidev_workspace
npx slidev slides.md
```

打开终端输出的本地地址，通常是 `http://localhost:3030`。看单 Agent 时，回到项目目录并将复制源改为 `single_round2_slides.md`，然后重新启动或刷新 Slidev。

## 下次重新启动：照此操作

```bash
# 1. 进入项目并激活已有环境
cd /Users/luwangqiang/Desktop/ai-agent-book/chapter5/paper-to-ppt
source ../../.venv/bin/activate

# 2. 可选：先检查渲染链路
python demo.py --smoke

# 3. 正式运行；改用新目录保留本次 r2 结果
python demo.py --mode both --max-rounds 2 \
  --out-dir validation/runs/qwen-both-r2-next

# 4. 查看汇总与最终图片
python -m json.tool validation/runs/qwen-both-r2-next/comparison_summary.json
open validation/runs/qwen-both-r2-next/rendered/
```

若 `.venv`、Node 依赖或 `.env` 不在，先按上文“项目位置与环境”“Qwen API 配置”恢复。只跑一种方案时，把 `both` 换成 `dual` 或 `single`。

## 可选：Gradio 网页控制台

当前项目已有 `app.py`。激活环境并安装 `requirements.txt` 后，在项目目录运行：

```bash
python app.py
```

浏览器打开 `http://127.0.0.1:7860`，选择 `both / dual / single`、轮数和新的输出目录名，点击“开始实验”。网页会显示日志、汇总指标及产物链接；它调用的仍是同一个 `demo.py`，需要相同的 `.env` 和渲染依赖。若命令提示缺少 Gradio，执行 `python -m pip install -r requirements.txt`。网页会拒绝使用已有非空输出目录，避免覆盖旧实验。
