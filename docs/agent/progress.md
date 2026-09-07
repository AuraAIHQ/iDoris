# iDoris 统一模型服务 实时状态 — progress

> 「此刻仓库真实发生了什么」。由 `pilot run` 每一步更新。
> 更新时间：2026-09-07 09:20

## 当前聚焦
- **Milestone**：M1 统一网关 MVP
- **Feature**：F1.1 契约层落地
- **正在开发的 Task**：无（规划刚落地，尚未开工任何 task）
- **分支 / worktree**：`docs/agent-ledger` / `../iDoris-plan`（规划专属 worktree，不属于任何 Feature）
- **PR**：本台账自身待开 PR 进 `preview`

## 仓库基线（2026-09-07 盘点）
- 集成分支 `preview` 已建立并推送；`main` 有 active ruleset 保护，只由 `preview` 经受控 PR 进入。
- PR #3（十篇规划文档 + U0 spike log）已 squash 合并进 `preview`；本地分支 `docs/idoris-unified-model-plan` 已清理。
- **仓库尚无任何代码**——只有 `docs/`（规划）与 `spike/u0/`（实测日志）。M1 的第一个 task 就是起 pnpm workspace 骨架。
- 遗留分支 `test/cla-action-check`：对应 PR #1 已 CLOSED **未合并**，safe-cleanup 正确地不动它，是否废弃待人决定。

## 进行中 / 待回执的 PR
| Task | PR | 状态 | 备注 |
|:---|:---|:---|:---|
| —（规划台账）| 待开 | — | `docs/agent-ledger` → `preview` |
| —（跨仓库需求）| [#4](https://github.com/iDoris-ai/iDoris/pull/4) | PR_OPEN | iDoris-website 提的 R0–R6；**base 是 `main`，需改成 `preview`**；且分支基于 PR #3 合并前的点，diff 里混进了 01–10 全部文档，真实新增只有 `docs/11-来自Starter-Kit的需求.md` |

## 阻塞项（BLOCKED）

- **R0 · iDoris Router 的归属与多租户语义**（来自 iDoris-website PR #4，需 @jhfnetboy 拍板）
  - **问题**：`iDoris-website` 的 `products/gateway/`（意图路由 + 成本闸 + 审计留痕）与本仓库 `01 §3` 的 iDoris Router **是同一层**，目前两套并行且没有任何文档写明关系。
  - **结构性错配**：`01` 把 iDoris 定位为「**个人** AI 网关」（能力①硬编码 loopback + 单用户，社区/城市端禁转发订阅）；泰国业务是**托管多客户**，需要 `tenant` 维度的用量/预算/审计隔离。多租户只涉及能力②③，**不触碰能力①的订阅红线**。
  - **两种答案**：(a) 归 iDoris → website 侧把 `products/gateway/` 降级为消费者，`routing.py` / `audit.py` / `egress_guard.py`（同为 Apache-2.0，含变异测试）整体移交；(b) 不归 iDoris → website 保留为泰国业务独立组件，两侧文档写明适用边界。
  - **影响范围**：选 (a) 会给 M1 增加一个「部署模式 = personal | tenant」的维度，波及 T1.3.2（policy）、T1.4.2（egress 门禁）、T2.2.3（审计）；选 (b) 则 M1 完全不受影响。**在拍板前不动这三个 task 的多租户设计**。
- **T2.5.2 三路由保真度矩阵**：缺 Anthropic + Gemini API key（用户凭证）；且无真实消费者。解除条件见 [`tasks.md`](tasks.md) 该 task。
- **T3.4.1 DP-FedLoRA**：当前写不出可机器验证的验收命令，保持 BACKLOG 不进 READY，等 F3.3 跑通后细化。

## 待人拍板的其余问题（不阻塞 M1 开工）
- PR #4 的 R1–R6（预算硬停 = 拒绝而非降级 / 审计只存元数据 / 账期时区显式 / 出网启动断言等）—— 其中 R4「审计只存元数据、绝不存内容」与本台账 T2.2.3 的设计**已经一致**；R1「预算是商业约束，不该走技术性降级链」需要在 `routing_policy` 里区分 `budget_exceeded`（终态拒绝）与 `oom/timeout`（可降级），是对 [`spec.md`](spec.md) 状态机的合理补充。待 R0 定了归属再一并处理。
- `test/cla-action-check` 分支（PR #1 已关闭未合并）是否废弃。

## 最近完成
- 2026-09-07 建立集成分支 `preview`；PR #3 retarget 到 `preview` 并 squash 合并（`dd9ae99`）；清理已合并本地分支 `docs/idoris-unified-model-plan`。
- 2026-09-07 落地 `docs/agent/` 规划七件套（research / acceptance / architecture / spec / roadmap / tasks / progress）+ `.pilot.yml`，把 `docs/01~10` 十篇散文规划蒸馏为 M→F→T 三级台账。

## 下一个 READY
- **T1.1.1** pnpm workspace 骨架 + 门禁流水线（无依赖）
- **T1.1.2** 五份契约的 TS 类型 + zod schema（依赖 T1.1.1）
- **T1.1.3** 组件卡策略校验器（依赖 T1.1.2）
