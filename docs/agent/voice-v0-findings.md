# Voice V0 现状盘点 — 回复泰国 Starter Kit

> 对方 `starter-kit/PRODUCT-FORM-AND-ROADMAP.md` 的 **V0** 里程碑写着「**需读 iDoris 代码仓库**（不在本仓库）」，
> 是整条 Voice 线的硬阻塞，也是 Dev 入职第一周第一件事。这份是回复。
> 核查日期：2026-09-07 ｜ 核查范围：`~/Dev/auraai/*`、`~/Dev/tools/*`

## 0. 结论先行：假设成立，但你们找错了仓库

> **「Voice 已有基础」这个假设是真的 —— 但它不在 iDoris，也不在 Agent24，在 [`iDoris-ai/AgentEar`](https://github.com/iDoris-ai/AgentEar)（本地 `~/Dev/tools/AgentEar`）。**

- **iDoris 仓库**：24 个文件，**零代码**，只有规划文档 + 一份 U0 spike 日志。没有任何语音实现。
- **Agent24**：`agent24-worker` crate 只有 **Rust 侧的 wire 契约与 HTTP 客户端**（定义了 `POST /v1/transcribe`）。**Python 侧 ML worker 未实现**，`docs/specs/TASKS.md` 标 `D4b deferred（等消费者）`。
- **AgentEar**：**M1 完成、M2 完成**（v0.4.2），有发布版 `.app`，实测一次 7.4 秒录音**全程 7.8 秒**完成，空闲常驻内存 13 MB。

## 1. 逐条回答 V0 的四问

| V0 问 | 答 |
|:---|:---|
| **哪个 whisper 实现与模型尺寸** | **不是 whisper。** 主链路是 **SenseVoiceSmall q8 GGUF**（~234M 参数 / ~250MB），跑在 **FunASR llamacpp runtime**（`runtime-llamacpp-v0.1.9`，macos-arm64）+ `fsmn-vad.gguf` 做 VAD。经四模型实测横比选定（`docs/decisions/0001-asr-model-selection.md`），初版基于二手资料的选型结论已被实测推翻并作废。 |
| **泰语单独调过没** | **调过，而且是单独一条引擎。** SenseVoice **做不了泰语**——实测两次，它把泰语音频标成 `<\|en\|>` 并输出乱码；其语种标记集里**根本没有 `th`**。故泰语走**独立的 whisper.cpp 引擎 + 显式语言选择**（不是自动检测），决策见 `docs/decisions/0004-thai-asr-engine.md`，模型指纹在 ADR 里固定、由 app 按需下载。另有泰语语料归档与实测结论（commit `2013c51`）。 |
| **有没有可跑的 demo** | **有。** GitHub Releases 提供 `AgentEar-*-macos-arm64.zip`，拖进「应用程序」即用；按右 Command 录音、再按停止，几百毫秒后转写进剪贴板。另有 `agentear --diagnose` 环境自检、`--transcribe x.wav` 离线转写。 |
| **部署形态** | **macOS `.app`，Apple Silicon 限定（M1 及以上，macOS 11+）。** |

## 2. 三条会改变你们排期的事实

### 2.1 ⚠️ `faster-whisper 停更 9 个月` 这条风险**不适用**，但换了一条

你们 `verification-2026-09-06-voice-stack.md` 的结论是「faster-whisper 停更 9 个月而后端仍在发版」。**AgentEar 根本没用 faster-whisper**，用的是 FunASR/SenseVoice + llamacpp runtime。

**那条风险作废，但要换成新的**：主链路依赖 `modelscope/FunASR` 的 `runtime-llamacpp` 发布产物，且 `docs/asr-selection.md` 记着 SenseVoice-Small **官方标注「即将停止维护」**。风险形状变了，不是消失了 —— 建议你们把核查项改写而不是删掉。

### 2.2 🔴 Apple Silicon 限定，与「Voice 可单独部署在客户机器上」冲突

你们 `starter-kit/README.md` §5 写的例外是「数据敏感度高的客户，Voice 可以单独部署在他们机器上」。

**但 AgentEar 不支持 Intel，也不支持 Windows/Linux** —— 而且这不是「暂时没做」：上游 FunASR 的 macOS 产物从 `v0.1.9` 到 `v0.2.6` **只有 `macos-arm64`**（Linux/Windows 才有 x64）。就算主程序编成通用二进制，Intel 上 ASR 子进程照样起不来。

**所以「部署到客户机器」这个承诺，目前只对用 Apple Silicon Mac 的客户成立。** 清迈的小生意用什么机器，这是你们比我清楚的事 —— 但它得先被知道。

### 2.3 AgentEar 的理解层**已经在消费一个 OpenAI-compat 本地端点**

`src/correct.rs` / `src/label.rs` 默认连 `http://127.0.0.1:8793`，发 `POST /v1/chat/completions`，用的模型是 **Ornith**（提示词里专门处理了它默认吐 `Thinking Process:` 的行为）。

**这正是 iDoris 该顶上的位置。** 现在 README 要求用户「自己备一个本地 LLM」；接上 iDoris 之后，那一句可以变成「指向你的 iDoris URL」，并且顺带获得隐私路由、预算、审计。

## 3. 对你们的建议

1. **V0 判定为「已完成」**，答案在本文档 §1，不必再等 iDoris —— 你们的 V1（锁版本 + GPU 路径验证）可以直接开工，但对象是 AgentEar 的 `vendor/bin` 产物版本，不是 faster-whisper × ctranslate2 那一对。
2. **V2 泰语评测基线仍然要做，而且更要做** —— AgentEar 有泰语引擎但主链路 SenseVoice 明确不覆盖泰语，你们要的「20 段真实泰语音频、字符错误率有数字」在 AgentEar 侧也还没有。这条是你们和 AgentEar 都受益的交付物。
3. **先确认客户机器架构**，再决定 §2.2 那个承诺怎么措辞。
