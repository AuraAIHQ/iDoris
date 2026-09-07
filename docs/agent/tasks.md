# iDoris 统一模型服务 · 任务台账 — Task

> 前置：[`roadmap.md`](roadmap.md)（M→F）· [`architecture.md`](architecture.md) · [`spec.md`](spec.md) · [`acceptance.md`](acceptance.md)
> 每个 Task 自包含，可独立开发与验收。**验收标准必须可机器验证**（跑命令能判定）。
> 状态：BACKLOG · READY · IN_PROGRESS · BLOCKED · PR_OPEN · CHANGES_REQUESTED · APPROVED · DONE
> 记录日期：2026-09-07

---

## F1.1 — 契约层落地

### T1.1.1 pnpm workspace 骨架 + 门禁流水线  `READY`
- **优先级**：high
- **目标**：起一个能跑 lint/typecheck/build/test 的 TS monorepo，后续所有 task 有落点。
- **开发范围**：`pnpm-workspace.yaml`；`packages/contracts` `packages/router` 两个空包；tsconfig（strict）、eslint、vitest、`.github/workflows/ci.yml` 跑同一套门禁。
- **明确不做**：不写任何业务逻辑；不引入 oMLX/HTTP 依赖；不配置发布流程。
- **依赖**：无
- **交付物**：`pnpm-workspace.yaml`、`package.json`（scripts: lint/typecheck/build/test）、两个包的骨架、CI workflow
- **验收命令**：`pnpm install && pnpm lint && pnpm typecheck && pnpm build && pnpm test`（全部退出 0；test 允许 0 用例但脚本必须存在且成功）
- **涉及文件**：仓库根、`packages/contracts/`、`packages/router/`、`.github/workflows/ci.yml`
- **风险/回滚**：无（纯新增）
- **证据**：<Branch / PR / 合并 commit>

### T1.1.2 五份契约的 TS 类型 + zod schema  `READY`
- **优先级**：high
- **目标**：把 06 §10 的契约片段变成可执行、可校验的类型。
- **开发范围**：`ProviderDescriptor` / `ComponentCard` / `LoadPolicy` / `RoutingPolicy` / `TaskProfile` 五份的 TS 类型与 zod schema，字段与取值严格按 [`spec.md`](spec.md) 数据模型章节。
- **明确不做**：不做 `AdapterManifest`（M3）；不做校验器的错误信息本地化；不实现任何解释/执行逻辑。
- **依赖**：T1.1.1
- **交付物**：`packages/contracts/src/{provider,component-card,load-policy,routing-policy,task-profile}.ts` + 导出的 zod schema
- **验收命令**：`pnpm --filter @idoris/contracts test`（每份契约至少 1 条合法样例通过 + 1 条缺必填字段样例被拒绝）
- **涉及文件**：`packages/contracts/`
- **风险/回滚**：契约变更影响所有下游 → 本 task 内定版为 v1，后续改动走版本化
- **证据**：<…>

### T1.1.3 组件卡策略校验器（缺字段即拒绝注册）  `READY`
- **优先级**：high
- **目标**：让「协议兼容 ≠ 路由安全」成为代码保证——缺策略字段的组件卡进不了 registry。
- **开发范围**：`validateComponentCard()`：强制 `privacy_class` / `allowed_egress` / `fallback_policy` / `fail_closed` / `version_pin` 存在；**交叉规则**：`privacy_class=local_only` 必须 `fail_closed=true`；`tier=local` 且 `locality=remote` 为非法；`allowed_egress` 含 `internet` 时 `privacy_class` 不得为 `local_only`。
- **明确不做**：不做 registry 本身（T1.3.1）；不做运行时 health 探测。
- **依赖**：T1.1.2
- **交付物**：`packages/contracts/src/validate.ts` + 非法样例表
- **验收命令**：`pnpm --filter @idoris/contracts test:contract`（每条交叉规则至少 1 条反例被拒绝，断言错误类型而非仅断言抛错）
- **涉及文件**：`packages/contracts/src/validate.ts`
- **风险/回滚**：**涉安全**——校验器漏判等于隐私门禁失效；反例测试是唯一凭证，不得跳过
- **证据**：<…>

### T1.1.4 契约成熟度标注与 README  `BACKLOG`
- **优先级**：low
- **目标**：按 06 §10.10 诚实标注每份契约当前是 L0/L1/L2/L3，不假装可替换。
- **开发范围**：`packages/contracts/README.md` 成熟度表 + 每份 schema 顶部注释标注级别
- **依赖**：T1.1.2
- **验收命令**：`test -f packages/contracts/README.md && grep -qE 'L[0-3]' packages/contracts/README.md`
- **证据**：<…>

---

## F1.2 — 能力③ 本地模型编排

### T1.2.1 LoadPolicy 抽象接口 + mock 适配器  `BACKLOG`
- **优先级**：high
- **目标**：先定义引擎无关的 `ModelBackend` 接口并用 mock 实现，保证 Router 不依赖任何具体引擎。
- **开发范围**：接口 `list() / load(id) / unload(id) / status() / chat(req)`；mock 适配器维护内存中的 loaded 集合 + 可配置内存上限，模拟 `ok/soft/hard/ceiling` 压力分级与 LRU 驱逐。
- **明确不做**：不碰 oMLX；不做真实推理。
- **依赖**：T1.1.2
- **交付物**：`packages/adapters/src/backend.ts`、`packages/adapters/mock/`
- **验收命令**：`pnpm --filter @idoris/adapters test`（断言：pinned 模型在 ceiling 压力下不被驱逐；unpinned 按 LRU 驱逐；`admission` 正确返回 `coexist|requires_eviction`）
- **涉及文件**：`packages/adapters/`
- **风险/回滚**：无
- **证据**：<…>

### T1.2.2 oMLX 适配器  `BACKLOG`
- **优先级**：high
- **目标**：把 LoadPolicy 抽象映射到 oMLX v0.4.3 的真实 knob（U0 已实测全部端点）。
- **开发范围**：`mode:resident → model_settings.is_pinned=true`；`on_demand → unpinned`；`evict_to_load → 依赖 ProcessMemoryEnforcer`；显式 `POST /v1/models/{id}/load` `/unload`；`admission` 读 `GET /api/status` 的 `model_memory_max` 与 loaded 列表。
- **明确不做**：不封装 oMLX 的 `/v1/embeddings` `/v1/rerank` `/v1/responses`（M3 再说）；不处理 oMLX 升级（手动换 .app，用户操作）。
- **依赖**：T1.2.1
- **交付物**：`packages/adapters/omlx/`
- **验收命令**：`pnpm --filter @idoris/adapters test:integration`（本机有 oMLX 时真起 `omlx serve --memory-guard balanced --memory-guard-gb 16` 跑 load→warm-hit→evict 序列；**无 oMLX 时必须打印 SKIPPED 并以非零以外方式明示跳过，不得静默通过**）
- **涉及文件**：`packages/adapters/omlx/`
- **风险/回滚**：v0.4.3 已知小限制——VLM 引擎 guard 传播告警（`could not resolve scheduler for VLMBatchedEngine`），不阻塞，记 followup
- **证据**：<…>

### T1.2.3 跨平台后端探测骨架  `BACKLOG`
- **优先级**：mid
- **目标**：启动时按 OS/硬件选后端，**Router 核心不出现 `omlx` 字样**。
- **开发范围**：`detectBackend()`：macOS+Apple Silicon → oMLX；Win/Linux+NVIDIA → vLLM 槽位；否则 llama.cpp/Ollama 槽位。非 macOS 的实现可先是 `NotImplemented` 占位，但**接口必须齐**。
- **明确不做**：不实现 vLLM/llama.cpp 适配器本体（等真实平台需求）。
- **依赖**：T1.2.2
- **交付物**：`packages/adapters/src/detect.ts`
- **验收命令**：`pnpm --filter @idoris/adapters test` + `! grep -rn "omlx" packages/router/src`（Router 核心零引用即通过）
- **涉及文件**：`packages/adapters/src/detect.ts`
- **风险/回滚**：无
- **证据**：<…>

### T1.2.4 LoadPolicy 黄金一致性测试（L3）  `BACKLOG`
- **优先级**：mid
- **目标**：让「可替换」从承诺变成可验证的凭证——同一组序列打 mock 与 oMLX 语义等价。
- **开发范围**：一份共享测试套件，参数化跑在两个适配器上。
- **依赖**：T1.2.2、T1.2.1
- **验收命令**：`pnpm test:golden`
- **证据**：<…>

---

## F1.3 — iDoris Router 薄编排层

### T1.3.1 Router 骨架：HTTP 服务 + ProviderRegistry + `/v1/models`  `BACKLOG`
- **优先级**：high
- **目标**：起一个监听 `127.0.0.1:PORT` 的进程，从 `config/components/*.yaml` 加载组件卡（经 T1.1.3 校验）并暴露 `/v1/models`。
- **开发范围**：HTTP server；组件卡加载 + 校验 + registry；`/v1/models` 聚合各 backend 的模型清单；health/cooldown（连续 3 次失败进 30s cooldown）。
- **明确不做**：不做路由决策（T1.3.2）；不监听非 loopback 地址。
- **依赖**：T1.1.3、T1.2.1
- **交付物**：`packages/router/src/{server,registry,health}.ts`、`config/components/omlx.yaml`
- **验收命令**：`pnpm --filter @idoris/router test && pnpm smoke`（`curl -s localhost:$PORT/v1/models | jq -e '.data|length>0'`；且注入一张缺 `privacy_class` 的组件卡时**启动失败并退出非 0**）
- **涉及文件**：`packages/router/`、`config/components/`
- **风险/回滚**：**涉安全**——绑定地址必须硬编码 loopback，测试断言不监听 `0.0.0.0`
- **证据**：<…>

### T1.3.2 控制面 header 解析 + routing policy 解释引擎  `BACKLOG`
- **优先级**：high
- **目标**：意图/隐私/复杂度走 header 不走 prompt；路由规则是 YAML 数据不是代码。
- **开发范围**：解析 `X-iDoris-Privacy/Intent/Complexity/Capabilities/Fallback` → `TaskProfile`（**缺省 privacy = `local_only`**，保守默认）；加载 `config/routing-policy.yaml`，按序匹配首条命中，`default` 必填；产出候选 provider 列表。
- **明确不做**：不做语义自动识别意图（M2/T2.4.1）；不把策略逻辑写进代码分支。
- **依赖**：T1.3.1
- **交付物**：`packages/router/src/{profile,policy}.ts`、`config/routing-policy.yaml`
- **验收命令**：`pnpm --filter @idoris/router test`（覆盖：header 缺省 → local_only；非法值 → 400；规则按序首条命中；无 `default` 的 policy 文件加载失败）
- **涉及文件**：`packages/router/src/`、`config/routing-policy.yaml`
- **风险/回滚**：策略文件版本化，破坏性改动升 `version`
- **证据**：<…>

### T1.3.3 fail-closed 隐私门禁 + 降级链  `BACKLOG`
- **优先级**：high
- **目标**：`local_only` 任务在本地不可用时**报错而非外泄**——这是产品承诺的落地点。
- **开发范围**：按 [`spec.md`](spec.md) 的请求路由状态机实现：`NO_CANDIDATE` 时若 `fail_closed` → 终态 503 `local_only_unavailable`；否则按 `next_in_chain` 走降级，且降级候选必须通过 `privacy_class` 与 `allowed_egress` 复核（**不信任 policy，二次校验**）。
- **明确不做**：不做重试（T1.3.4）；不做审计落盘（M2/T2.2.3）。
- **依赖**：T1.3.2
- **交付物**：`packages/router/src/dispatch.ts`
- **验收命令**：`pnpm test:privacy` —— 停掉全部本地 provider，发 20 条 `local_only` 请求，断言 **20 条全部 503 且假上游出站计数器 == 0**；任一条出站即失败
- **涉及文件**：`packages/router/src/dispatch.ts`
- **风险/回滚**：**阻断级**——此测试失败不接受「已知问题」标注，必须修到过
- **证据**：<…>

### T1.3.4 `/v1/chat/completions` 转发：streaming、重试、幂等、取消  `BACKLOG`
- **优先级**：high
- **目标**：让标准 openai SDK 不改代码即可调通，且失败行为可预期。
- **开发范围**：非流式 + SSE 流式转发；非流式最多 2 次退避重试（250ms→1s + jitter），**流式已吐 token 后不重试**，以 SSE error 事件终止；可选 `X-iDoris-Request-Id` 幂等键（60s 窗口）；客户端断开时向上游传播取消。
- **明确不做**：不做 tool-calling 的语义翻译（能力②相关，延后）；不做多模态输入。
- **依赖**：T1.3.3
- **交付物**：`packages/router/src/proxy.ts`
- **验收命令**：`pnpm smoke`（用 openai SDK 跑非流式 + 流式各一次，断言首 token 到达；断开连接后断言上游收到 abort）
- **涉及文件**：`packages/router/src/proxy.ts`
- **风险/回滚**：流式重试会导致重复 token —— 测试显式断言「已吐 token 后不重试」
- **证据**：<…>

### T1.3.5 驱逐竞态互斥锁  `BACKLOG`
- **优先级**：mid
- **目标**：防止两个请求同时驱逐彼此需要的模型形成活锁。
- **开发范围**：per-backend 互斥锁，`evict_to_load` 全程持锁；等待超时 10s → 返回 `oom` 而非无限等待。
- **依赖**：T1.3.4、T1.2.1
- **验收命令**：`pnpm --filter @idoris/router test`（并发 10 个需要互相驱逐的请求，断言无死锁、无超过 10s 的等待、全部有终态）
- **证据**：<…>

### T1.3.6 出网启动断言（启动期部署配置，与运行期路由是两个洞）  `BACKLOG`
- **优先级**：mid
- **目标**：证明 Router 进程在**处理任何请求之前**不会自己出网——`test:privacy` 测的是运行期路由，这条测的是启动期部署配置，两者抓不到对方的问题。
- **开发范围**：socket 层打桩拒绝所有非本机连接，启动 Router 后断言零出网；**必须配正对照**：一个刻意出网的用例要被探针抓到——抓不到出网的探针，它报的「零出网」什么都不证明。
- **明确不做**：不做运行期出站管控（那是 T1.4.2 的 egress-guard）。
- **依赖**：T1.3.1
- **交付物**：`packages/router/test/egress-probe.ts`（含正对照用例）
- **验收命令**：`pnpm test:egress`（零出网断言通过 **且** 正对照用例确实被探针捕获；正对照不红即判本 task 未完成）
- **涉及文件**：`packages/router/test/`
- **风险/回滚**：**涉隐私**——参考实测：某些库不设变量时零出网，但设了 tracing 类环境变量后会连外部端点。风险不在库，在部署配置
- **证据**：<…>

---

## F1.4 — 能力① 订阅中转（可选 / best-effort）

### T1.4.1 subprocess 中转适配器  `BACKLOG`
- **优先级**：mid
- **目标**：把已登录的 `claude` / `codex` 订阅态封成 OpenAI-compat provider（U0 已验证 `claude -p` 与 `codex exec` 可行）。
- **开发范围**：spawn CLI → 解析输出 → 封 OpenAI-compat 响应；120s 超时；取消/超时走 `SIGTERM → 5s → SIGKILL`，**不留孤儿进程**。
- **明确不做**：不做 streaming（CLI 非交互模式先按整块返回）；不做 OAuth 路径（CLIProxyAPI 模式，备选）。
- **依赖**：T1.3.1
- **交付物**：`packages/adapters/subscription/`
- **验收命令**：`pnpm --filter @idoris/adapters test:integration`（本机有 `claude` 时断言 `claude -p "reply with exactly: IDORIS_RELAY_OK"` 经网关返回该字符串；进程清理断言 `ps` 无残留子进程；无 CLI 时打印 SKIPPED）
- **涉及文件**：`packages/adapters/subscription/`
- **风险/回滚**：孤儿进程会吃满机器 —— 清理断言是硬性验收项
- **证据**：<…>

### T1.4.2 loopback + 单用户门禁（合规红线落地）  `BACKLOG`
- **优先级**：high
- **目标**：让「社区端/城市端绝不转发个人订阅」成为代码约束而非文档劝告。
- **开发范围**：订阅 provider 的组件卡固定 `allowed_egress: [loopback]`；Router 在 dispatch 前复核请求来源为 loopback（或用户显式开启的 Tailscale 私网白名单）；部署模式为 `community|city` 时该 provider **拒绝注册并明确报错**。
- **明确不做**：不做多租户鉴权（不在本轮范围）。
- **依赖**：T1.4.1、T1.1.3
- **交付物**：`packages/router/src/egress-guard.ts`、`config/components/subscription.yaml`
- **验收命令**：`pnpm --filter @idoris/router test`（断言：非 loopback 来源的订阅请求被拒；`IDORIS_DEPLOY_MODE=community` 时启动即拒绝注册订阅 provider 并退出非 0）
- **涉及文件**：`packages/router/src/egress-guard.ts`
- **风险/回滚**：**涉合规**——此门禁是能力①得以存在的前提，测试不得跳过
- **证据**：<…>

### T1.4.3 订阅中转不作必需 fallback 的断言  `BACKLOG`
- **优先级**：mid
- **目标**：落实 U0 决策 S3——能力①是 best-effort，核心正确性不依赖它。
- **开发范围**：routing policy 中订阅 provider 永不出现在 `default` 链；测试断言禁用订阅 provider 后所有非订阅场景仍全绿。
- **依赖**：T1.4.2
- **验收命令**：`IDORIS_DISABLE_SUBSCRIPTION=1 pnpm test`（全绿）
- **证据**：<…>

---

## F2.1 — HardwareAwareModelRecommender

### T2.1.1 硬件探测 + 内存公式  `BACKLOG`
- **优先级**：high
- **目标**：实现 07 §1–§3 的公式：`footprint = params × bpp + KV(ctx) + 开销`，Apple 预算三档。
- **开发范围**：`system_profiler`/`sysctl`/`os.cpus` 探测 `ram_gb/chip/gpu_cores`；量化 bpp 与质量表；KV 估算（层数×KV头×head_dim×ctx×kv_quant）。
- **明确不做**：不做推荐决策（T2.1.2）。
- **依赖**：T1.1.2
- **交付物**：`packages/recommender/src/{probe,memory}.ts`
- **验收命令**：`pnpm --filter @idoris/recommender test`（对照 U0 实测值断言：Qwen3-8B 4bit ≈4.48GB、VL-7B ≈8.50GB、8B 的 KV ≈9.00MB/64token，误差 <5%）
- **涉及文件**：`packages/recommender/`
- **风险/回滚**：公式偏差会导致 OOM —— 用 U0 实测数据作回归基线
- **证据**：<…>

### T2.1.2 打分与推荐算法 + `IDORIS_CORE_MODEL` override  `BACKLOG`
- **优先级**：high
- **目标**：按 07 §5.3 伪代码选出常驻 + 临时组合。
- **开发范围**：`catalog.yaml` 加载；常驻选 `capability.reasoning × quant.quality` 最高且放得下的；临时按需求能力逐个 admission；不下则标 `requires_eviction`；`IDORIS_CORE_MODEL` 强制时推荐模块让路但仍输出警告。
- **依赖**：T2.1.1
- **交付物**：`packages/recommender/src/recommend.ts`、`config/catalog.yaml`
- **验收命令**：`pnpm --filter @idoris/recommender test`（断言 M4/24GB profile 输出 `resident=ornith-1.0-9b@q6_k` 且 `agents-a1-35b` 判 `BLOCKED`；16GB 降到 q4/q5；32GB 允许 35B）
- **风险/回滚**：目录数据错误会推荐出跑不动的组合 —— catalog 每条附 `min_ram_gb` 硬门槛
- **证据**：<…>

### T2.1.3 可读 tradeoff 输出 + sysctl 建议  `BACKLOG`
- **优先级**：mid
- **目标**：推荐不是黑箱打分，要能解释为什么这么选。
- **依赖**：T2.1.2
- **验收命令**：`pnpm --filter @idoris/recommender test`（断言输出含 `warnings[]`、`recommended_sysctl.iogpu_wired_limit_mb`、非空 `tradeoff` 文本）
- **证据**：<…>

---

## F2.2 — 容量接口与可观测

### T2.2.1 `GET /capabilities`  `BACKLOG`
- **优先级**：high
- **目标**：把容量变成接口（06 §10.8），业务据此知道能否共存/要不要驱逐。
- **开发范围**：每个能力附 `resident/estimated_memory_gb/ctx_limit/queue_depth/admission_status(ready|requires_eviction|blocked)`；数据源为 recommender + backend `status()`。
- **依赖**：T2.1.2、T1.3.1
- **验收命令**：`pnpm smoke`（`curl /capabilities | jq -e '.[]|select(.admission_status)'` 非空且取值在枚举内）
- **证据**：<…>

### T2.2.2 native 特性收进 extensions 命名空间  `BACKLOG`
- **优先级**：low
- **目标**：防 provider 锁定（06 §10.7）——原生特性不得裸用，必须带降级声明。
- **依赖**：T1.1.2
- **验收命令**：`pnpm --filter @idoris/contracts test:contract`（无 `_degradation` 声明的 extension 被拒绝）
- **证据**：<…>

### T2.2.3 路由决策审计日志  `BACKLOG`
- **优先级**：mid
- **目标**：每次路由留下「选了谁、为什么、是否降级」的记录（acceptance 第二节「可审计」）。
- **开发范围**：结构化日志：`request_id / profile / matched_rule / candidates / chosen / degraded / latency`。**两道防线**（来自 iDoris-website 实现，Apache-2.0 可直接移植）：① 写入前对记录的**字段名**逐个比对黑名单（`prompt/prompts/input/content/text/body/messages/document/file/payload` 等 frozenset），命中即抛 `ContentLeakError` **拒绝写入**——不是静默丢弃（静默丢弃会让人以为内容被存下来了）；② 单字段 500 字符上限——长文本出现在元数据里，本身就是「有人把内容塞进来了」的信号。
- **明确不做**：**绝不记录请求或响应内容**（仅元数据）。
- **依赖**：T1.3.3
- **验收命令**：`pnpm test:audit`（① 哨兵字符串不出现在日志；② 含黑名单字段名的记录**抛错**而非被清洗后写入；③ 超 500 字符的字段被拒绝。对照 iDoris-website 的两条变异测试：「字段名不再比对禁用清单」「取消 500 字符上限」，改坏后必须变红）
- **风险/回滚**：**涉隐私**——内容入日志等于隐私承诺作废，哨兵测试是硬性验收项
- **证据**：<…>

---

## F2.3 — Agent24 集成交接

### T2.3.1 交出 R1–R6 需求并确认接口  `BACKLOG`
- **优先级**：mid
- **目标**：按 [`../09-对Agent24的需求.md`](../09-对Agent24的需求.md) 与 Agent24 侧确认 `IDORIS_URL` provider 接入方式（纯加法零回归）。
- **明确不做**：**不改 Agent24 内核代码**——iDoris 只提需求。
- **依赖**：T1.3.4
- **验收命令**：`test -f docs/agent/handoff-agent24.md`（含 R1–R6 逐条的接口约定与联调命令）
- **证据**：<…>

### T2.3.2 端到端联调冒烟  `BACKLOG`
- **优先级**：mid
- **目标**：Agent24 把 `IDORIS_URL` 指过来后原有功能零回归且能用到本地模型。
- **依赖**：T2.3.1
- **验收命令**：`pnpm smoke:agent24`（Agent24 provider 列表出现 iDoris；一次 `local_only` 调用落到本地模型）
- **证据**：<…>

---

## F2.4 — 语义意图路由

### T2.4.1 引入 semantic-router 做意图识别  `BACKLOG`
- **优先级**：low
- **目标**：把 `X-iDoris-Intent` 从「调用方必须声明」升级为「未声明时可自动识别」（[`../10-入口路由模型-调研对比.md`](../10-入口路由模型-调研对比.md) 首选方案）。
- **明确不做**：不替代 header —— **显式声明永远优先于识别结果**；隐私字段绝不自动推断。
- **依赖**：T1.3.2
- **验收命令**：`pnpm --filter @idoris/router test`（断言：显式 header 存在时识别结果被忽略；`privacy` 永不被自动推断）
- **证据**：<…>

---

## F2.5 — 能力② 外部 API 槽位

### T2.5.1 OpenAI-compat 上游槽位 + 冒烟  `BACKLOG`
- **优先级**：low
- **目标**：只证明「外部槽位可接」，不做选型。
- **开发范围**：通用 OpenAI-compat 上游适配器；用 `.env` 的 `OPENAI_API_KEY` 做一次冒烟。
- **明确不做**：不做 Anthropic/Gemini 翻译；不引入 LiteLLM/ClawRouter/OmniRoute。
- **依赖**：T1.3.4
- **验收命令**：`pnpm smoke:external`（有 key 时调通一次；无 key 时打印 SKIPPED 而非失败）
- **证据**：<…>

### T2.5.2 三路由保真度矩阵（OmniRoute / ClawRouter / LiteLLM）  `BLOCKED`
- **优先级**：low
- **目标**：产出「provider × 特性 × 是否统一可用」矩阵，作为能力②的验收基线。
- **阻塞原因**：① 需要 Anthropic + Gemini API key（用户凭证，尚未提供）；② 尚无真实消费者 —— 按「先有消费者再有提供者」原则不提前做。
- **解除条件**：用户提供两把 key，**或**明确「只测 ClawRouter 免费层 + 本地模型」并接受结论范围受限。
- **依赖**：T2.5.1
- **验收命令**：`test -f docs/agent/fidelity-matrix.md`（含 streaming / tool-calling / 多模态 / thinking 四项 × 各 provider）
- **证据**：<…>

---

## F3.x — 自增长与联邦（M3，全部 BACKLOG）

> 严格按 03 §5 的 F0 → F1 → F2 推进。**硬门禁：F3.4 隐私层就位前，真实个人数据不得进入联邦；F3.1/F3.3 只用合成/脱敏数据。**

### T3.1.1 本地数据湖 + 提炼管线（合成数据）  `BACKLOG`
- **优先级**：low ｜ **依赖**：T2.2.3 ｜ **验收命令**：`pnpm --filter @idoris/growth test`（合成语料跑通 使用→数据湖→提炼 三段，断言产出的训练样本 schema 合法且**不含真实数据标记**）

### T3.1.2 MLX-LoRA 本地训练 + 热挂载  `BACKLOG`
- **优先级**：low ｜ **依赖**：T3.1.1、T3.2.1 ｜ **风险**：python 3.9.6 可能不满足 mlx-lm（需 3.10+），U0 已记 ｜ **验收命令**：`pnpm --filter @idoris/growth test:integration`（训练出一个 rank=16 的 adapter 并经 Router 热挂载后可推理；无 MLX 环境打印 SKIPPED）

### T3.2.1 AdapterManifest + base 一致性门禁  `BACKLOG`
- **优先级**：low ｜ **依赖**：T1.1.2 ｜ **风险**：**涉正确性**——指纹校验失效会「一次静默升级毁掉整批 LoRA」 ｜ **验收命令**：`pnpm --filter @idoris/contracts test:contract`（断言 base/tokenizer digest 不匹配的 manifest 被拒绝挂载与聚合）

### T3.3.1 Flower 最小联邦（同 base、只传 LoRA、FedAvg）  `BACKLOG`
- **优先级**：low ｜ **依赖**：T3.1.2、T3.2.1 ｜ **验收命令**：`pnpm --filter @idoris/federation test:integration`（两个本地客户端 + 合成数据跑通一轮聚合；断言传输载荷**只含 adapter 权重、不含原始样本**）

### T3.4.1 DP-FedLoRA 加噪 + 安全聚合  `BACKLOG`
- **优先级**：low ｜ **依赖**：T3.3.1 ｜ **验收命令**：待 F3.3 完成后细化（当前写不出可机器验证的命令，故保持 BACKLOG 不进 READY）

### T3.4.2 真实数据准入门禁  `BACKLOG`
- **优先级**：low ｜ **依赖**：T3.4.1 ｜ **验收命令**：`pnpm --filter @idoris/federation test`（断言隐私层未启用时，标记为真实数据的样本**无法**进入联邦管线，报错而非跳过）

---

## 跟进项账本（followups）

| # | 来源 | 内容 | 状态 |
|:---|:---|:---|:---|
| FU-1 | U0 实测 | oMLX v0.4.3 VLM 引擎 guard 传播告警（`could not resolve scheduler for VLMBatchedEngine`），不阻塞，等新版 .app | OPEN |
| FU-2 | U0 环境探测 | 本机 python 3.9.6，mlx-lm 训练可能需 3.10+，M3 开工前处理 | OPEN |
| FU-3 | 05 §8 第 5 条 | 凭证网关（onecli 式 MITM）2026-09-07 拍板记 BACKLOG；architecture 已留 `CredentialProvider` 抽象位 | OPEN |
| FU-4 | 跨仓库 | iDoris-website PR #4 `docs/11-来自Starter-Kit的需求.md`：R0 网关归属 + 多租户语义待拍板，见 [`progress.md`](progress.md) 阻塞项 | OPEN |
| FU-5 | 跨仓库 R6 | 账期/时区：若将来引入计费，时区必须显式，且**回归测试要真的切换进程时区**（`TZ` + `tzset()`）跑多个时区同一套断言——只在测试内部造时间戳不动 TZ 的写法，会让 bug 在任何时区都自洽地变绿。本仓库当前无计费，暂不落 task | OPEN |
| FU-6 | 跨仓库 R1 | `budget_exceeded` 已作为与 `local_only_unavailable` 同级的终态写进 [`spec.md`](spec.md) 状态机；对应的 routing policy 字段等 R0 定了归属再落 task | OPEN |
