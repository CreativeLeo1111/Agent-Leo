# Experiment 5-7：Paper-to-Video（Qwen 适配版）

> 基于 `ai-agent-book/chapter5/paper-to-video` 的论文讲解视频实验。  
> 本实验在原始 Experiment 5-6 基础上完成 Qwen 适配，并进一步打通 Experiment 5-6 的真实论文 PPT 输出，实现：

```text
论文 PDF
→ Paper-to-PPT
→ 真实 Slidev PPT
→ Qwen Plus 生成逐页讲解词
→ Qwen3-TTS-Flash 生成中文语音
→ ffmpeg 逐页音画同步
→ 最终论文讲解视频
```

---

## 1. 实验目标

Experiment 5-7 的目标是将论文幻灯片、口语化讲解词和语音放到同一条时间线上，自动生成带旁白的论文讲解视频。

本次实验重点完成以下目标：

1. 跑通原始 `paper-to-video` 的离线视频合成链路；
2. 将文本讲解模型替换为 `qwen-plus`；
3. 将 TTS 替换为 DashScope 的 `qwen3-tts-flash`；
4. 将 Experiment 5-6 中真实生成的 19 页 Slidev PPT 接入视频生成流程；
5. 使用 ffmpeg 保证每页 PPT 展示时间与对应语音时长一致；
6. 最终生成完整论文讲解视频。

---

## 2. 与 Experiment 5-6 的衔接

本实验直接使用 Experiment 5-6 的正式 Qwen 对照实验产物。

使用的 PPT Markdown：

```text
../paper-to-ppt/validation/runs/qwen-both-r2/dual_round2_slides.md
```

使用的真实渲染页面：

```text
../paper-to-ppt/validation/runs/qwen-both-r2/rendered/dual_round2/
```

该目录包含：

```text
1.png
2.png
3.png
...
19.png
```

检查结果：

```text
Slidev Markdown 页面数：19
PNG 页面数：19
```

因此 Markdown 源码与真实渲染页面可以一一对应。

---

# 3. 下次如何重新启动

这是以后最常用的部分。

## 3.1 进入项目目录

```bash
cd /Users/luwangqiang/Desktop/ai-agent-book/chapter5/paper-to-video
```

## 3.2 激活虚拟环境

```bash
source ../../.venv/bin/activate
```

正常情况下终端前面会显示：

```text
(agentbook)
```

## 3.3 运行真实 PPT → 视频

```bash
python real_ppt_video.py
```

运行完成后，最终视频位于：

```text
output/real_ppt/lecture.mp4
```

直接打开：

```bash
open output/real_ppt/lecture.mp4
```

查看讲解词：

```bash
cat output/real_ppt/narration.json
```

检查最终视频信息：

```bash
ffprobe -v error \
-show_entries stream=codec_name,codec_type,width,height \
-show_entries format=duration \
-of default=noprint_wrappers=1 \
output/real_ppt/lecture.mp4
```

---

# 4. 实验环境

本实验环境：

```text
设备：MacBook Air M1
Python：3.12.14
虚拟环境：agentbook
项目：bojieli/ai-agent-book
目录：chapter5/paper-to-video
```

DashScope SDK：

```text
dashscope==1.27.7
```

确认版本：

```bash
python -c "import dashscope; print(dashscope.__version__)"
```

输出：

```text
1.27.7
```

视频工具：

```text
ffmpeg
ffprobe
```

---

# 5. 模型配置

本实验使用阿里云 DashScope。

`.env` 配置：

```env
OPENAI_API_KEY=<YOUR_DASHSCOPE_API_KEY>
OPENAI_BASE_URL=https://dashscope.aliyuncs.com/compatible-mode/v1

TEXT_MODEL=qwen-plus

TTS_MODEL=qwen3-tts-flash
TTS_VOICE=Cherry
```

> 不要将真实 API Key 上传到 GitHub。

建议确保 `.env` 已加入 `.gitignore`。

---

# 6. 与原始 Experiment 5-5 的区别

原仓库正式 `campaign.py` 使用的方案主要包括：

```text
Narration：Kimi K3
Visual Reviewer：Qwen-VL-Max
TTS：Fish Audio S1
Video：ffmpeg
```

而本次实验采用：

```text
Narration：Qwen Plus
TTS：Qwen3-TTS-Flash
Video：ffmpeg
PPT：Experiment 5-4 中实际生成的 Slidev PPT
```

因此，本实验属于：

> **Experiment 5-5 的 Qwen 适配与扩展版本**

它不是对原官方 Kimi + Fish Audio 配置的完全原样复现。

---

# 7. 原始项目环境检查

首先进入：

```bash
cd /Users/luwangqiang/Desktop/ai-agent-book/chapter5/paper-to-video
```

运行：

```bash
python demo.py --check
```

该命令不会调用 API，主要检查：

- ffmpeg
- ffprobe
- 中文字体
- API 配置
- TEXT_MODEL
- TTS_MODEL
- TTS_VOICE

---

# 8. 离线流水线验证

为了避免直接消耗 API，首先运行：

```bash
python demo.py --offline
```

离线模式使用静音占位音轨，验证以下链路：

```text
PIL 幻灯片
→ 静音音频
→ ffmpeg 单页视频
→ concat
→ lecture.mp4
```

产物包括：

```text
output/slides/
output/audio/
output/segments/
output/narration.json
output/lecture.mp4
```

查看视频：

```bash
open output/lecture.mp4
```

这一阶段的目的只是验证视频合成与逐页时长控制，不代表真实 TTS 效果。

---

# 9. Qwen Plus 文本讲解测试

首先验证 DashScope OpenAI-compatible 文本接口。

最初使用：

```python
load_dotenv()
```

配合：

```bash
python - <<'PY'
...
PY
```

出现：

```text
AssertionError
```

错误位置来自 `python-dotenv` 的自动 `.env` 查找机制。

解决方式：

```python
load_dotenv(".env")
```

之后通过：

```python
client = OpenAI(
    api_key=os.getenv("OPENAI_API_KEY"),
    base_url=os.getenv("OPENAI_BASE_URL"),
)
```

调用：

```text
qwen-plus
```

成功生成中文论文讲解词。

因此确认：

```text
.env
→ DashScope OpenAI-compatible API
→ qwen-plus
→ 中文口语化论文讲解词
```

链路正常。

---

# 10. DashScope SDK 安装

在当前虚拟环境中直接运行：

```bash
pip install -U dashscope
```

出现：

```text
zsh: command not found: pip
```

因此改用：

```bash
python -m pip install -U dashscope
```

若环境没有 pip，也可以使用：

```bash
uv pip install -U dashscope
```

最终确认：

```bash
python -c "import dashscope; print(dashscope.__version__)"
```

输出：

```text
1.27.7
```

---

# 11. Qwen3-TTS 最小测试

使用：

```text
qwen3-tts-flash
```

音色：

```text
Cherry
```

测试调用成功。

返回结果：

```text
status_code: 200
```

并得到：

```text
output.audio.url
```

该 URL 指向真实生成的 WAV 音频。

因此确认：

```text
中文文本
→ qwen3-tts-flash
→ Cherry
→ WAV
```

TTS 链路正常。

---

# 12. Qwen TTS 本地音频测试

通过返回的：

```text
response["output"]["audio"]["url"]
```

下载 WAV：

```python
urllib.request.urlretrieve(audio_url, output_file)
```

测试音频保存到：

```text
output/audio/qwen_tts_test.wav
```

播放：

```bash
open output/audio/qwen_tts_test.wav
```

确认真实中文语音正常。

---

# 13. demo.py 的 Qwen TTS 适配

原 `demo.py` 支持：

```text
openai
offline
```

本次增加：

```text
qwen
```

目标调用形式：

```bash
python demo.py --quick --tts-provider qwen
```

主要改动思路：

```text
synthesize_speech()
├── offline → synthesize_offline()
├── qwen    → synthesize_qwen()
└── openai  → synthesize_openai()
```

Qwen TTS 使用：

```python
dashscope.MultiModalConversation.call(
    model="qwen3-tts-flash",
    text=text,
    voice="Cherry",
)
```

随后下载返回的：

```text
audio.url
```

保存为本地 WAV。

---

# 14. 为什么没有继续使用内置 5 页 Demo

原 `demo.py` 自带的是 5 页 PIL 模拟幻灯片。

其主要价值是教学和快速验证：

```text
内置论文要点
→ PIL 生成 PPT
→ LLM
→ TTS
→ ffmpeg
```

但本项目已经完成 Experiment 5-4：

```text
真实论文
→ Agent
→ Slidev PPT
```

因此 Experiment 5-5 更合理的做法是直接消费 Experiment 5-4 的真实产物。

最终采用：

```text
dual_round2_slides.md
+
rendered/dual_round2/*.png
```

作为视频输入。

---

# 15. 真实 PPT 输入验证

PNG 文件：

```bash
find ../paper-to-ppt/validation/runs/qwen-both-r2/rendered/dual_round2 \
  -type f -name "*.png" | sort
```

实际存在：

```text
1.png
2.png
...
19.png
```

Markdown 检查：

```python
from pathlib import Path

p = Path(
    "../paper-to-ppt/validation/runs/qwen-both-r2/"
    "dual_round2_slides.md"
)

text = p.read_text(encoding="utf-8")

parts = [
    x.strip()
    for x in text.split("\n---\n")
    if x.strip() and "theme:" not in x
]

print("Slidev 页面数:", len(parts))
```

结果：

```text
Slidev 页面数: 19
```

最终确认：

```text
19 Markdown pages
=
19 rendered PNGs
```

---

# 16. real_ppt_video.py

为了避免破坏原始 `demo.py` 的教学流程，新建：

```text
real_ppt_video.py
```

该脚本专门负责真实 PPT → 视频。

核心流程：

```text
dual_round2_slides.md
        ↓
按 "---" 拆分 19 个页面
        ↓
对应读取 1.png ~ 19.png
        ↓
Qwen Plus
逐页生成中文讲解
        ↓
Qwen3-TTS-Flash
生成每页真实语音
        ↓
ffprobe
读取每页音频时长
        ↓
ffmpeg
PPT PNG + 对应音频
        ↓
seg_01.mp4 ... seg_19.mp4
        ↓
ffmpeg concat
        ↓
lecture.mp4
```

---

# 17. 讲解词生成策略

每页使用真实 Slidev Markdown 作为模型输入。

要求包括：

- 不逐条照读 PPT；
- 解释内容为什么重要；
- 与前后页有自然过渡；
- 页面存在公式、模型结构、数字或图时进行适当说明；
- 不虚构页面之外的数据；
- 使用自然中文讲课风格；
- 每页约 120～180 个中文字符。

开场页、中间页和最后一页使用不同提示。

---

# 18. 音画同步策略

每页流程：

```text
PNG
+
对应 TTS 音频
```

首先通过：

```bash
ffprobe
```

获得音频真实时长。

随后：

```bash
ffmpeg
```

使用：

```text
-t <audio_seconds>
```

将当前 PPT 页面的视频时长锁定为当前音频时长。

最后通过：

```text
concat
```

将 19 个页面片段拼接。

因此不是人为给每页固定 10 秒或 20 秒，而是：

```text
页面展示时间
=
该页真实旁白时间
```

---

# 19. 正式实验结果

完整 19 页实验运行成功。

终端最终输出：

```text
======================================
生成完成
======================================
页面数量: 19
音频总时长: 549.84s
最终视频时长: 549.98s
```

结果汇总：

| 指标 | 结果 |
|---|---:|
| PPT 页面数量 | 19 |
| 音频总时长 | 549.84 s |
| 最终视频时长 | 549.98 s |
| 音视频总时长差 | ≈ 0.14 s |
| 最终视频长度 | ≈ 9 分 10 秒 |

最终视频：

```text
output/real_ppt/lecture.mp4
```

讲解词：

```text
output/real_ppt/narration.json
```

---

# 20. 输出目录

最终主要结构：

```text
paper-to-video/
├── real_ppt_video.py
├── output/
│   └── real_ppt/
│       ├── lecture.mp4
│       ├── narration.json
│       ├── audio/
│       │   ├── audio_01.wav
│       │   ├── audio_02.wav
│       │   └── ...
│       └── segments/
│           ├── seg_01.mp4
│           ├── seg_02.mp4
│           └── ...
```

---

# 21. 最终视频检查

播放：

```bash
open output/real_ppt/lecture.mp4
```

检查编码、分辨率和时长：

```bash
ffprobe -v error \
-show_entries stream=codec_name,codec_type,width,height \
-show_entries format=duration \
-of default=noprint_wrappers=1 \
output/real_ppt/lecture.mp4
```

重点确认：

```text
codec_type=video
codec_name=h264
```

以及：

```text
codec_type=audio
codec_name=aac
```

---

# 22. 人工质量检查建议

除了程序上的时长一致，还应该人工检查：

### 第 1 页

检查：

- 是否自然开场；
- 是否正确介绍论文主题；
- 是否避免机械照读 bullet。

### 技术核心页面

重点检查：

- Self-Attention；
- Multi-Head Attention；
- Transformer Architecture；
- 公式和模型结构。

旁白应该解释：

```text
“为什么这样设计”
```

而不仅仅是：

```text
“页面上写了什么”
```

### Figure 页面

检查讲解是否与当前真正显示的 Figure 一致。

### 第 19 页

检查：

- 是否自然总结论文贡献；
- 是否形成完整的视频收尾。

---

# 23. 常见问题

## 23.1 `pip: command not found`

使用：

```bash
python -m pip install ...
```

或者：

```bash
uv pip install ...
```

---

## 23.2 `load_dotenv()` 出现 AssertionError

在：

```bash
python - <<'PY'
```

这类 stdin Python 运行方式下，不要使用：

```python
load_dotenv()
```

改成：

```python
load_dotenv(".env")
```

---

## 23.3 qwen-plus 能用，但 TTS 不能用

原因：

```text
qwen-plus
```

通过 DashScope 的 OpenAI-compatible 接口调用。

而：

```text
qwen3-tts-flash
```

本实验使用 DashScope SDK：

```python
dashscope.MultiModalConversation.call(...)
```

两者接口不同。

---

## 23.4 PPT 页面顺序错误

不要直接字符串排序：

```text
1.png
10.png
11.png
2.png
```

应该按数字排序：

```python
sorted(
    paths,
    key=lambda x: int(x.stem),
)
```

最终得到：

```text
1.png
2.png
...
19.png
```

---

## 23.5 Markdown 页面数与 PNG 不一致

必须先检查：

```text
Markdown pages == PNG pages
```

否则 Markdown 与画面可能错位。

---

## 23.6 不要上传 `.env`

`.env` 内包含 API Key。

检查 `.gitignore`：

```text
.env
```

必须被忽略。

---

# 24. 当前完整 Agent 工作流

目前已经打通：

```text
                  Paper
                    │
                    ▼
             PDF 内容提取
                    │
                    ▼
          Experiment 5-4
             Paper-to-PPT
                    │
          ┌─────────┴─────────┐
          │                   │
     Dual Agent          Single Agent
          │                   │
          └─────────┬─────────┘
                    │
             独立视觉评分
                    │
                    ▼
             最终 Slidev PPT
                    │
                    ▼
          Experiment 5-5
            Paper-to-Video
                    │
              Qwen Plus
                    │
              中文讲解词
                    │
                    ▼
          Qwen3-TTS-Flash
                    │
                WAV 音频
                    │
                    ▼
                ffmpeg
                    │
                    ▼
             Lecture Video
```

---

# 25. 实验结论

本次 Experiment 5-5 Qwen 适配成功完成了从真实论文 PPT 到论文讲解视频的完整自动化链路。

最终系统不再只是一个内置 5 页 Demo，而是能够消费 Experiment 5-4 真实生成的 19 页 PPT，并自动完成：

```text
理解页面
→ 编写逐页讲解
→ 中文语音生成
→ 页面与语音同步
→ 视频拼接
```

最终得到约：

```text
9 分 10 秒
```

的完整论文讲解视频。

音频总时长：

```text
549.84 s
```

最终视频时长：

```text
549.98 s
```

二者仅相差约：

```text
0.14 s
```

说明基于真实音频时长控制 PPT 页面展示时间的方案能够实现较好的全局时间对齐。

---

# 26. 下一步扩展

目前输入仍然指向固定的：

```text
qwen-both-r2
```

后续可以继续改造成真正通用的应用：

```text
上传任意论文 PDF
        ↓
自动生成 PPT
        ↓
用户预览 / 选择最终 PPT
        ↓
自动生成逐页讲解词
        ↓
Qwen3-TTS
        ↓
自动生成论文讲解视频
        ↓
网页直接播放
```

进一步可加入：

- Gradio Web UI；
- 任意 PDF 上传；
- 自动选择 Dual / Single Agent；
- PPT 在线预览；
- 视频在线预览；
- 讲解词人工修改后重新 TTS；
- 多种音色切换；
- 中英文视频；
- 单页重新生成；
- 失败页面续跑；
- 视觉 Agent 审核旁白与 PPT 的一致性；
- 自动生成实验指标与 GitHub 报告。

这样可以把 Experiment 5-4 与 5-5 从两个独立实验升级为一个完整的：

> **Paper → Slides → Narration → Speech → Video Agent System**

