# iDoris 统一模型服务 · 架构 — 技术判断与骨架

> 「怎么搭」。定义契约与**不可动摇的边界**。数据细节见 [`spec.md`](spec.md)。
> 完整论证见 [`../01-统一模型服务-架构设计.md`](../01-统一模型服务-架构设计.md)、[`../05-集成与技术栈协调规划.md`](../05-集成与技术栈协调规划.md)、[`../06-组件接口契约与互换标准.md`](../06-组件接口契约与互换标准.md)。
> 记录日期：2026-09-07

## 核心判断

1. **集成契约 = REST(OpenAI-compat HTTP) + 进程边界，不是 import、不是同语言。**
   生态本就是 polyglot（Agent24 Rust+TS、agent-speaker Go、auraai-packages TS）。强求同语言不可能也没必要。进程边界同时买到四样东西：稳定（组件崩溃不波及）、许可干净（不 vendor 源码即可合法集成 GPL 上游）、可换（契约后随便替）、可取消可审批（跨进程天然可 kill）。与 Agent24 SPEC-001 §10「进程边界 = 授权边界」同源。

2. **能力③ 用 oMLX，不装 llama-swap（U0 实测后的决策变更）。**
   原计划用 llama-swap 做「常驻/临时 + 一个 URL + 内存受控」，探测发现 `omlx serve` 本身就是 LRU-based 多模型服务器，原生具备且实测通过（8B 热命中 0.39s、16GB guard 下精确核算、模型级驱逐实测确认、内置 KV 估算与 07 §1 公式一致）。**纯 MLX 场景下 llama-swap 是多余一层**——anti-over-engineering。

3. **但契约不写死 oMLX。** oMLX 仅 Apple Silicon，而 Agent24 要分发 Windows/Linux/macOS。故 capability③ 的实现**按平台探测选择**，全部在 `LoadPolicy` 抽象之后。oMLX 是「macOS 的默认实现」，不是 iDoris 的固定依赖。

4. **local-first 的能力优先级**：`③ 本地模型(核心) > ① 订阅中转(可选/best-effort) > ② 外部 API(罕用逃生口)`。
   能力①**绝不**作为核心正确性的必需 fallback；能力②这一轮只留 provider 槽位 + OpenAI-compat 冒烟。

5. **策略是数据不是代码。** 路由/隐私/fallback 逻辑若硬编码进 Router，换 Router 就要移植隐藏策略。故 `routing_policy` 是版本化的声明式 YAML，Router 只是**解释引擎**。

6. **Router 第一版 TypeScript / pnpm**（2026-09-07 拍板）。与 auraai-packages / iDoris-SDK 同栈，oMLX 与 CLI 中转都是 HTTP/subprocess，TS 起最快。验证后按 06 契约内化进 Agent24 Rust `ModelRouter`——**契约不变，实现可换**这条对我们自己也成立。

## 系统骨架

```
              ┌──────── 对外：唯一入口 http://127.0.0.1:PORT/v1 (OpenAI-compat) ────────┐
 Agent24 /    │  + 控制面 header  X-iDoris-Privacy / Intent / Complexity / Capabilities │
 渠道 / 业务 ─▶└──────────────────────────────┬──────────────────────────────────────────┘
                                ┌─────────────▼──────────────┐
                                │   iDoris Router (TS 进程)   │
                                │  ┌──────────────────────┐  │
                                │  │ 控制面解析 → TaskProfile│  │
                                │  │ routing_policy 解释引擎 │  │  ← 策略是 YAML，不是代码
                                │  │ ProviderRegistry(组件卡)│  │  ← 缺策略字段拒绝注册
                                │  │ fail_closed 隐私门禁    │  │  ← LocalOnly 无本地可用即报错
                                │  │ 降级链 / 用量 / 审计    │  │
                                │  └──────────────────────┘  │
                                └──┬───────────┬──────────┬───┘
                     REST ─────────┘    REST   │  subprocess+REST └────────┐
                       ▼                       ▼                            ▼
        ┌──────────────────────────┐ ┌────────────────────┐ ┌────────────────────────────┐
        │ 能力③ 本地模型（核心）    │ │ 能力② 外部 API      │ │ 能力① 订阅中转（可选）      │
        │ LoadPolicy 适配器:        │ │ (槽位，本轮只冒烟)  │ │ spawn `claude -p`/`codex   │
        │  macOS → oMLX :8088      │ │ OpenAI-compat 上游  │ │ exec` → 封 OpenAI-compat   │
        │  Win/Linux → vLLM/       │ │ 选型 BLOCKED        │ │ loopback + 单用户 硬绑定    │
        │   llama.cpp/Ollama       │ └────────────────────┘ └────────────────────────────┘
        └──────────────────────────┘
                                    ┌──────────────────────────────────┐
                                    │ M2: HardwareAwareModelRecommender │  ← 读 /api/status 做 admission
                                    │ M3: 联邦训练进程(Flower+PEFT+mlx) │  ← 离线批处理，按需拉起
                                    └──────────────────────────────────┘
```

## 契约 / 接口（实现与调用分离）

四份契约是**本项目的核心资产**，组件是可替换耗材。完整字段见 [`spec.md`](spec.md)，语义来源见 06 §10。

| 契约 | 作用 | 成熟度目标 |
|:---|:---|:---|
| `ProviderDescriptor` | 统一 provider 家族/tier/capability/privacy_class/locality 五套词汇 | M1 达 L1 骨架 → L2 完整 schema |
| `ComponentCard` | 组件注册单元，**强制**含 `privacy_class` `allowed_egress` `fallback_policy` `fail_closed` | M1 L1 + 校验器 |
| `LoadPolicy` / `ModelLease` | 抽象「常驻/临时/驱逐载入」，引擎无关 | M1 L1 + oMLX 适配 → M2 L3 黄金测试 |
| `RoutingPolicy` | 声明式路由规则（if privacy/intent/complexity → then tiers/capability/fail_closed）| M1 L1 |
| `AdapterManifest` | LoRA 的 base/tokenizer 指纹 + framework + 隐私处理 + 聚合兼容性 | M3 |

**控制面（06 §10.5）**：意图/隐私/复杂度**不走 prompt、不走 model 名**，走扩展 header（对 OpenAI-compat 透明）：
```
X-iDoris-Privacy: local_only | any
X-iDoris-Intent: banner | blog | reasoning | coding | chat
X-iDoris-Complexity: simple | complex
X-iDoris-Capabilities: vision,asr
X-iDoris-Fallback: fail_closed | next_in_chain
```
备选方式 B：侧端点 `POST /idoris/route` 返回选定 provider 后再调 `/v1`。

**契约成熟度分级（06 §10.10）**：L0 命名 → L1 骨架（强制字段+语义）→ L2 完整 I/O schema → L3 黄金一致性测试。**「可替换」这个承诺只对达到 L2+L3 的能力成立**；未达的诚实标注，不假装。

## 不可动摇的边界

- **LocalOnly fail-closed**：`privacy_class: local_only` 的任务，本地无可用 provider 时**报错**，绝不降级到 loopback 以外的任何目的地。这是产品承诺，不是最佳实践。
- **组件卡缺策略字段即拒绝注册**：协议兼容 ≠ 路由安全。没有 `privacy_class`/`allowed_egress`/`fallback_policy`/`fail_closed` 的组件不进 registry。
- **能力①只绑 loopback + 单用户**：社区端/城市端配置下**代码层面无法启用**订阅中转（不是文档劝告）。用户可显式放开到 Tailscale 私网，属个人自用延伸。
- **不 vendor 第三方源码**：二进制/CLI/容器引入，一切经 pin 的版本（二进制版本号 / 容器 tag / commit / 模型指纹），杜绝悄悄升级破坏兼容。需 patch 的协议层才用 submodule 固定 commit。
- **不写死单一推理后端**：任何 `omlx` 字样只能出现在 `adapters/omlx/` 下；Router 核心只认 LoadPolicy 契约。
- **base 指纹不匹配拒绝聚合/挂载**（M3）：防止一次静默升级毁掉整批 LoRA。
- **联邦真实数据门禁**（M3）：隐私层（DP-FedLoRA + 安全聚合）未就位时，真实个人数据不得进入联邦；F0 只用合成/脱敏数据。
- **战略平台依赖不抽象**：Nostr（去中心通信）与 AirAccount DID（身份信任根）是 Mycelium 的既定赌注，直接依赖，**不套可替换抽象层**——把它们当可换组件反而增加无谓复杂度（06 §10.9）。

## 运行形态

- **iDoris Router**：常驻 Node 进程（pnpm workspace），监听 `127.0.0.1:PORT`；测试期跑个人电脑，部署到 Mac mini 24h 常驻，经 Tailscale 从任意地点访问同址。
- **能力③ 后端**：独立进程（macOS 上是 oMLX .app / `omlx serve`），Router 通过 HTTP 调用与 `/load` `/unload` 显式控制。
- **能力① 中转**：按请求 spawn `claude -p` / `codex exec`，非常驻。
- **联邦训练**（M3）：Python 进程，离线批处理，按需拉起，不常驻。
- **跨仓库**：Agent24 把 iDoris 统一 URL 作为一个 provider 接入（`IDORIS_URL`，纯加法零回归）；需 Nostr 时 subprocess 驱动 agent-speaker(Go) CLI。生态 lingua franca = **OpenAI-compat HTTP + Nostr + DID**。
