# iDoris Router · 多租户接口契约 v1（对外）

> 面向 iDoris Router 的**消费方**（如 iDoris-website Starter Kit Gateway）。
> 状态：**契约 v1 冻结**（2026-09-07）。实现见 `tasks.md` 的 F1.5；契约先行，实现随后。
> 破坏性变更会升 `v2` 并保留 v1 至少一个里程碑。
> 上位文档：[`architecture.md`](architecture.md) 核心判断 7 · [`spec.md`](spec.md) TenantContext / 路由顺序 / 审计记录。

## 0. 背景与边界

R0 已拍板：**多租户属于 iDoris 的范围。iDoris 是组织大脑，未来为组织提供服务。**
消费方不再自行维护平行的路由/成本/审计层，改为调用本契约。

**Router 只管三件事**：哪个模型、能不能调、记什么账。
**Router 明确不管**：消费方的业务语义（业务动作、字数规则、审批队列）、前端、租户的身份签发。

## 1. 部署形态

```
deploy_mode: personal | tenant
```

| | `personal` | `tenant`（组织大脑）|
|:---|:---|:---|
| `X-iDoris-Tenant` | 忽略 | **必填**；缺失 → `400 tenant_missing`，**不回落默认租户** |
| 能力① 订阅中转 | 可用（loopback + 单用户）| **拒绝注册，启动即报错退出非 0** |
| 能力②③ | 可用 | 可用，按 tenant 隔离用量/预算/审计 |

> **回答消费方的猜测：是的，`deploy_mode=tenant` 下能力①直接不可用，这里写死。**
> 不是运行时拒绝单个请求，而是**启动时拒绝注册该 provider**——组织租户用组织自己的 API（能力②）或本地模型（能力③）。loopback + 单用户红线不因多租户松动；这两件事正交。

## 2. 调用面：怎么带 tenant

沿用既有的 `X-iDoris-*` 控制面 header（对 OpenAI-compat 透明，消费方仍用标准 openai SDK）：

```http
POST /v1/chat/completions
X-iDoris-Tenant: acme-co            # tenant 模式必填
X-iDoris-Privacy: local_only | any  # 缺省 = local_only（保守默认）
X-iDoris-Intent: banner|blog|reasoning|coding|chat
X-iDoris-Complexity: simple|complex
X-iDoris-Capabilities: vision,asr
X-iDoris-Fallback: fail_closed|next_in_chain
```

**响应头回传决策**（无需额外查询即可解释单次调用）：
```http
X-iDoris-Reason: privacy_enforced | budget | intent_match | degraded
X-iDoris-Provider: <provider_id>
X-iDoris-Cost-Minor: <int>          # 本次调用成本，最小货币单位
```

## 3. TenantContext

```yaml
tenant_id: acme-co                  # 消费方负责验明身份后传入；Router 不做签发/鉴权
budget:
  limit_minor: 5000000              # 账期上限，最小货币单位（整数，避免浮点）
  spent_minor: 1234567              # 本账期已花费（Router 维护）
  scope: paid_only | all            # 默认 paid_only，见 §4
billing_timezone: Asia/Bangkok      # ★ 必填、显式 IANA 时区名，不得取服务器本地时区
quota:                              # 可选
  rpm: 60
  tpm: 200000
```

**`tenant_id` 的来源**：Router **不做租户鉴权与签发**——那是组织侧的事。Router 只消费「已验明的 tenant_id」，并对它做隔离与计账。消费方必须保证不把 A 租户的请求打上 B 的 tenant_id。

## 4. 预算：耗尽是**拒绝**，不是降级

预算闸门位于路由链的第 2 步（**隐私判定之后、意图匹配之前**）：再便宜的候选也是花钱，不该等选完模型才检查。

超限响应：
```http
HTTP/1.1 402 Payment Required
{"error": {"type": "budget_exceeded",
           "message": "租户 acme-co 本账期预算已用尽，这不是服务故障；请调整预算上限或等待下一账期。",
           "tenant_id": "acme-co", "limit_minor": 5000000, "spent_minor": 5000000}}
```
- **不产生任何计费调用**——超限时上游一次都不会被调到。
- 错误信息**必须让人分得清「是预算不是故障」**，否则客户会以为服务坏了。
- **绝不自动降级到便宜档**：客户设了上限就是不想再花钱，自动降级等于替他决定「继续花，只是花得少些」。

### `budget.scope` —— 本契约对上游需求的一处细化
上游原始需求是「预算硬停排在任务匹配之前」。对纯外部 API 的网关这是对的，但 **iDoris 有零成本的本地模型**，一刀切会让超预算租户连不花钱的本地推理都用不了。故：

| scope | 语义 |
|:---|:---|
| `paid_only`（默认）| 预算只闸住 `cost > 0` 的候选。超预算后**本地模型仍可用**（成本 0）|
| `all` | 组织要求「超预算就完全停」时使用，任何调用一律拒绝 |

两者都是**显式选择，不留给实现推断**。消费方若希望沿用「超预算完全停」的旧行为，请显式配 `scope: all`。

## 5. 隔离保证

用量 / 预算 / 审计三类数据的访问层**强制携带 tenant 作用域**：

- A 租户查不到 B 租户的**任何一条**记录。
- **缺 tenant 上下文的查询直接抛错**，而不是返回全量——靠调用方每次记得加 `where tenant_id = ?` 是失败开放。

## 6. 审计与用量查询（消费方的计费依据）

### 审计记录字段（穷举白名单）
`request_id` · `tenant_id` · `component` · `intent` · `privacy` · `tier` · `provider_id` · `model_id` · `tokens_in` · `tokens_out` · `cost_minor` · `latency_ms` · `status` · `reason` · `ts_utc`

- **`reason` 必须非空**，取值 `privacy_enforced` / `budget` / `intent_match` / `degraded`。一句 `routed` 不合格。
- **绝不记录内容**：prompt / completion / 任何客户数据。两道防线——字段名黑名单闸门（命中即**抛错拒绝写入**，不是静默丢弃）+ 单字段 500 字符上限。
- `ts_utc` 存 UTC epoch；**账期换算发生在聚合时**，用租户的 `billing_timezone`。

### 查询接口（M2 / F2.6 交付）
```http
GET /idoris/tenants/{tenant_id}/usage?period=2026-09     # 月度用量与成本
GET /idoris/tenants/{tenant_id}/budget                   # 余额
GET /idoris/tenants/{tenant_id}/audit?from=&to=&limit=   # 审计记录（仅元数据）
```

**`period` 的时区语义（计费依据，务必读）**：
`period=2026-09` **一律按该租户 `TenantContext.billing_timezone` 解释**，**绝不使用服务器本地时区**，也不接受调用方在查询里另指定时区（避免同一租户不同调用方切出不同的月）。

**响应必须回显边界，让调用方能验证而不是只能信任**：
```json
{
  "tenant_id": "acme-co",
  "period": "2026-09",
  "billing_timezone": "Asia/Bangkok",
  "range_utc": {"from": "2026-08-31T17:00:00Z", "to": "2026-09-30T17:00:00Z"},
  "totals": {"cost_minor": 1234567, "tokens_in": 0, "tokens_out": 0, "calls": 0}
}
```
`billing_timezone` 与 `range_utc` 是**响应的必填字段**。理由：这个坑（换台机器部署账单就变、且没有任何东西报错）的隐蔽之处在于**两边都没有可对账的凭据**。把解析出的 UTC 边界回显出来，调用方可以直接断言它，错了当场看得见。

**硬保证**：同一批数据在任何时区的机器上聚合，`totals` 与 `range_utc` 必须完全一致——不是尽力而为。

## 7. 消费方迁移清单

1. 把本地路由/成本/审计层改成薄客户端：调 `/v1/*` 并带 `X-iDoris-Tenant`。
2. 预算配置从本地搬到 `TenantContext`，**显式选定 `budget.scope`**（想保持旧的「超预算全停」行为就选 `all`）。
3. **显式设 `billing_timezone`**（如 `Asia/Bangkok`），不要依赖服务器时区。
4. 计费改读 §6 的用量查询接口。
5. 保留本地实现直到本契约的实现跑通——**避免出现「两边都没有」的窗口**（这条是消费方提的，采纳）。

## 8. 未定 / 后续

- **租户鉴权与签发**：不在本契约内，组织侧负责。
- **跨租户聚合报表**（组织管理员视角）：未定，需要时另起契约。
- **`quota.rpm/tpm` 限流**：字段已留位，语义与实现待 F1.5 之后。
