# GitHub Copilot

GitHub Copilot 是双轨数据源：**CLI 本地文件**（OTEL JSONL 需手动开启 + session-state events.jsonl 默认持久化）与**云端 REST API**（个人 premium-request/AI-credit 口径，企业 metrics NDJSON 报告）。IDE（VS Code/JetBrains 等）**不在本地落任何 token 级数据**，只能走 API。

数据质量评级：
- CLI session-state events.jsonl：**丰富**（五类 token + premium-request 数 + AIU 费率表，但仅会话级累计快照）
- CLI OTEL JSONL：**丰富但默认不存在**（事件级、可去重，需用户预先开启遥测）
- 个人 API（premium_request/usage）：**估算级**（请求数口径，无 token）
- 企业 metrics reports：**有限**（日粒度聚合，token 仅 prompt/output 两档求和，无成本）

> 本机验证说明：调研机器上 `~/.copilot/` 仅存在 `hooks/` 目录（无 `otel/`、无 `session-state/`），即未产生过 Copilot CLI 会话。以下本地格式基于 ccusage adapter 源码（`rust/adapters/copilot/`）与官方 GitHub 文档/OpenAPI 描述撰写，本地文件部分**未经本机验证**——token 语义裁决表（见下）的每一格须在 adapter 开工时用真实样本验证后冻结（golden fixtures，M0.5 冻结门）。

## 数据位置

| 数据 | Linux/macOS | Windows |
|---|---|---|
| OTEL 导出 | `~/.copilot/otel/*.jsonl` | `%USERPROFILE%\.copilot\otel\*.jsonl` |
| OTEL 显式路径 | `$COPILOT_OTEL_FILE_EXPORTER_PATH`（任意位置，单文件） | 同左 |
| 会话状态 | `~/.copilot/session-state/<session-uuid>/events.jsonl` | `%USERPROFILE%\.copilot\session-state\<uuid>\events.jsonl` |
| 云端 API | `https://api.github.com/...`（无本地文件） | 同左 |

OTEL 导出默认**关闭**，开启方式（用户侧前置条件，Tokly 不得代为设置）：

```bash
export COPILOT_OTEL_ENABLED=true
export COPILOT_OTEL_EXPORTER_TYPE=file
export COPILOT_OTEL_FILE_EXPORTER_PATH="$HOME/.copilot/otel/copilot-otel-$(date +%Y%m%d-%H%M%S).jsonl"
```

session-state 的 events.jsonl 由 CLI **默认写入**，无需任何开关（Copilot CLI ≥ 0.0.4xx 均如此），是大多数机器上唯一存在的本地数据。

## 格式详解

### A. OTEL JSONL（`~/.copilot/otel/*.jsonl`）

每行一条 OTEL span 或 log 记录（两种导出器序列化形态都出现过）。顶层字段：`traceId` / `spanId`（或嵌套 `spanContext.{traceId,spanId}`）、`name`、`type`（`"span"`）、`startTime`/`endTime`/`hrTime`（形态为 `[秒, 纳秒]` 数组或标量，需按数量级自适应：纳秒/微秒/毫秒/秒）、`timeUnixNano`、`timestamp`、`body`/`_body`、`attributes`（点键平铺对象）。

**token 字段确切位置**（全部在 `attributes` 内，值可能是数字或数字字符串）：

| Tokly 字段 | attributes 键（按优先级） |
|---|---|
| input | `gen_ai.usage.input_tokens` —— 包含关系按「token 语义裁决」节判定；净 input 公式见该节 |
| output | `gen_ai.usage.output_tokens` |
| cacheRead | `gen_ai.usage.cache_read.input_tokens` |
| cacheWrite5m | `gen_ai.usage.cache_write.input_tokens`，回退 `gen_ai.usage.cache_creation.input_tokens`（无 5m/1h 分档概念，全部计入 cacheWrite5m，`cacheWrite1h` 恒 0，ADR-0002） |
| reasoning | `gen_ai.usage.reasoning.output_tokens`，回退 `gen_ai.usage.reasoning_tokens` |
| total（包含关系判别用） | `gen_ai.usage.total_tokens`，回退 `gen_ai.usage.total.token_count` |

模型：`gen_ai.response.model` → `gen_ai.request.model` → 同 traceId 其他记录回填 → `null`（ADR-0002 禁魔法值）。注意 Copilot 内部模型名带后缀（如 `claude-opus-4.7-1m-internal`、`gpt-5.4-mini`），查价前需归一化。

会话归属（属性优先级从高到低）：`gen_ai.conversation.id` / `copilot_chat.session_id` / `copilot_chat.chat_session_id` / `session.id`（同级）→ `github.copilot.interaction_id` → `gen_ai.response.id` → 同 traceId 回填 → traceId 本身。**无项目/cwd 字段**。

ccusage 识别四类带 usage 的记录，按优先级互斥抑制（高优先级存在的 trace/responseId 下，丢弃低优先级记录）：

1. **chat span**：`gen_ai.operation.name == "chat"` 或 `name` 以 `"chat "` 开头
2. **inference log**：`event.name == "gen_ai.client.inference.operation.details"` 或 body 前缀 `"GenAI inference:"`
3. **agent turn log**：`event.name == "copilot_chat.agent.turn"`（含 `turn.index` / `copilot_chat.turn.index`）
4. **agent summary span**：`gen_ai.operation.name == "invoke_agent"`

**无原生成本字段**——OTEL 文件内没有任何价格信息，成本须由 LiteLLM 价格表计算。

虚构示例（chat span 单行，值均为编造）：

```json
{"traceId":"7f3a9c2e1b4d4e8f9a0b1c2d3e4f5a6b","spanId":"0a1b2c3d4e5f6071","name":"chat gpt-5.4","startTime":[1760000000,120000000],"endTime":[1760000003,840000000],"attributes":{"gen_ai.operation.name":"chat","gen_ai.response.model":"gpt-5.4","gen_ai.conversation.id":"conv-9f8e7d6c-1111-2222-3333-444455556666","gen_ai.response.id":"resp_abc123xyz","gen_ai.usage.input_tokens":41000,"gen_ai.usage.output_tokens":1200,"gen_ai.usage.cache_read.input_tokens":36000,"gen_ai.usage.cache_write.input_tokens":0}}
```

### B. session-state events.jsonl（默认数据）

`~/.copilot/session-state/<uuid>/events.jsonl`，每行 `{"type": ..., "timestamp": ISO8601, "data": {...}}`。Tokly 关心三类：

- `session.start` → `data.sessionId`、`data.copilotVersion`、`data.startTime`、`data.context.cwd`、`data.repository`（**项目归属的唯一来源**）
- `session.shutdown` → `data.modelMetrics.<model>.usage.{inputTokens, outputTokens, cacheReadTokens, cacheWriteTokens, reasoningTokens}` 与 `data.modelMetrics.<model>.requests.{count, cost}`。**会话级累计快照**，一次会话可能有多条 shutdown，取最后一条；`requests.cost` 是 premium-request 数（如 `15`），不是 USD。
- `session.compaction_complete` → `data.copilotUsage.tokenDetails[]`（每项 `{tokenType: input|cache_read|cache_write|output, batchSize, costPerBatch, tokenCount}`）与 `data.copilotUsage.totalNanoAiu`。费率为 nano-AIU（10⁻⁹ AI Unit）：`credits = totalNanoAiu / 1e9`，`USD = credits × 0.01`。这是 CLI 自带的**原生费率表**，≥ ~1.0.4x 版本才有。

虚构示例（shutdown 事件节选，值均为编造）：

```json
{"type":"session.shutdown","timestamp":"2026-07-20T08:15:30.000Z","data":{"modelMetrics":{"gpt-5.4":{"usage":{"inputTokens":58200,"outputTokens":3100,"cacheReadTokens":21400,"cacheWriteTokens":8700,"reasoningTokens":0},"requests":{"count":4,"cost":3.6}}}}}
```

### C. 个人 API（premium-request / AI-credit 口径）

```
GET /users/{username}/settings/billing/premium_request/usage?year=2026&month=7&day=15&model=&product=
GET /users/{username}/settings/billing/ai_credit/usage          # 同参数，AI Credit 口径（2026-06 起的新计费）
```

- 认证：fine-grained PAT 或 GitHub App user token，**"Plan" user permission (read)**；classic PAT 亦可（走自身账号）。
- 参数：`year`（默认当年）、`month`（默认当月）、`day`（可选）、`model`、`product`（模糊过滤）。**历史深度 24 个月**。
- 响应：`{timePeriod:{year,month?,day?}, user, usageItems[]}`，每项 `{product, sku, model, unitType, pricePerUnit, grossQuantity, grossAmount, discountQuantity, discountAmount, netQuantity, netAmount}`。
- **口径警告**：`unitType` 为 `"requests"`（premium request 数，`pricePerUnit` 官方示例 0.04 USD）或 `"ai-credits"`（0.01 USD/credit）。**没有 token 数，没有会话/项目维度**。month 粒度时无法细分到天。
- 若用户 Copilot 许可由组织/企业代管，其用量**不出现**在 user 级端点，须用 org/enterprise 端点。

### D. 企业/组织 metrics reports（NDJSON 下载）

```
GET /enterprises/{ent}/copilot/metrics/reports/enterprise-1-day?day=YYYY-MM-DD
GET /enterprises/{ent}/copilot/metrics/reports/enterprise-28-day/latest
GET /enterprises/{ent}/copilot/metrics/reports/users-1-day?day=...        # 用户级，Tokly 主要用这条
GET /orgs/{org}/copilot/metrics/reports/{organization|users|repos|user-teams}-1-day?day=...
```

- 认证：OAuth/PAT(classic) 需 `manage_billing:copilot` 或 `read:enterprise`；fine-grained token 需 **"Enterprise Copilot metrics" (read)**（org 级为 "Organization Copilot metrics" (read)，classic 需 `read:org`）。前置：企业策略 "Copilot usage metrics" 必须 Enabled。
- 返回 `download_links[]`（**签名 URL，限时过期**，须立即下载）+ `report_day`。文件为 **NDJSON**。
- 历史：数据自 **2025-10-10** 起可用，最长回溯 **1 年**；T+1 生成，当日数据不可得。
- 用户级每行字段：`day`、`user_login`、`user_id`、`ai_credits_used`、`used_chat/used_agent/used_cli/used_copilot_app`、`totals_by_cli.{prompt_count,request_count,session_count,token_usage.{prompt_tokens_sum,output_tokens_sum,avg_tokens_per_request}}`、`totals_by_copilot_app`（同构）、`totals_by_feature[]`、`totals_by_ide[]`、`totals_by_language_model[]`、`totals_by_model_feature[]`、`loc_*_sum`。**token 仅 prompt/output 两档，无 cache/reasoning 细分，无 USD 成本**。

## 归一化映射

| UsageEvent | OTEL (A) | session-state (B) | 个人 API (C) | 企业报告 (D) |
|---|---|---|---|---|
| source | `"copilot"` | `"copilot"` | `"copilot"` | `"copilot"`（二轮审查 §1 交叉一致性：api/metrics **降为 channel**，不再作独立 source） |
| channel / status | `channel='otel'` | `channel='session-state'` | `channel='api'` | `channel='metrics'`；四通道并存时按用户配置的唯一权威通道计 `status='counted'`，非权威通道入库但 `status='excluded'`（见「去重与已知坑」权威裁决条） |
| granularity | `'event'` | `'session'`（shutdown 为会话级快照）；`periodEnd` = shutdown 时刻（覆盖期终点，与 C/D 通道月末/24:00Z 同构——2026-08-04 裁决显式化） | `'month'`（`periodEnd` = 月末，带 day 参数时 `'day'`） | `'day'`（`periodEnd` = 当日 24:00Z） |
| **eventKey（去重键）** | chat span: `{traceId}:{spanId}`；log: `log:{traceId}:{spanId}`；agent-turn: `agent-turn:{traceId}:{turn.index}`；缺 id 时退化 `span:{sessionId}:{tsMs}:{行号}` 并标 `identityQuality='weak'` | `shutdown:{sessionId}:{model}`（幂等，重扫天然去重） | `api:premium:{user}:{period}:{model}:{sku}` | `metrics:{user_login}:{day}:{scope}`（scope=cli/app/feature） |
| revision | `originStream` = 文件相对路径；`sourcePosition` = 字节偏移 | 同左 | `originStream` = 端点+周期；`sourcePosition` = 拉取页序（整周期 staging 发布后才有意义，见增量摄取节） | 同 C |
| sessionKey | 属性优先链（见上），兜底 traceId | 目录 uuid（即 sessionId） | null | null |
| parentSessionKey | 无（`--resume` 复用同一 session 目录/会话 id） | 同左 | — | — |
| projectKey / projectLabel | **不可得** | label = `session.start` 的 `context.cwd` 原始值（2026-08-04 裁决定稿：label 与 projectKey = normalize(cwd) 必须同源，展示与聚合不得分裂）；`repository` **不映射**（如有价值属未来 detail，另议）；key 由 core 统一归一化 | null | null |
| timestamp | `endTime` 优先，多形态自适应解析为 epoch_ms；兜底文件 mtime | 事件 `timestamp`（shutdown 时刻，非请求时刻） | 周期起点（月/日 00:00Z） | `day` 00:00Z |
| model | `gen_ai.response.model`（需去内部后缀归一化） | modelMetrics 的键名 | `model` 字段 | `totals_by_model_feature[].model`（常为空数组） |
| agent | `''`（四通道均无智能体/子代理概念，见「agent 归一化规则」节，ADR-0018） | 同左 | 同左 | 同左 |
| tokens.input | **净 input 公式**（见「token 语义裁决」节） | 按版本表（同节） | **不可得**（写 0 + `tokenCoverage` 标注全维度不可得） | `prompt_tokens_sum`（日总和） |
| tokens.output | `output_tokens`（reasoning 包含关系按版本表） | 同左 | 不可得 | `output_tokens_sum` |
| tokens.cacheRead / cacheWrite5m | clamp 公式（同节；无 5m/1h 分档 → 全部计入 cacheWrite5m（默认档），`cacheWrite1h` 恒 0，ADR-0002） | 同左 | 不可得 | 不可得（写 0 + `tokenCoverage` 标注） |
| tokens.reasoning | `reasoning.output_tokens` | `reasoningTokens` | 不可得 | 不可得 |
| tokenQuality | `'native'` | `'native'` | `'native'`（无 token 维度） | `'native'` |
| costNativeMicroUSD | null | null（`requests.cost` 是 premium-request 数、`totalNanoAiu` 是 AIU——**数量口径进 `event_charges`，不折算进成本列**） | `round(netAmount × 1e6)` | null（`ai_credits_used` → `event_charges`） |
| nativeCostKind / nativeCostProvider | — | — | `'premium-requests'` 或 `'ai-credits'`（按 `unitType`）/ `'github-billing'` | — |
| pricingEligibility / displayPolicy | `'standard'` / `'computed-only'` | `'standard'` / `'computed-only'` | `'disabled'` / `'native-only'`（无 token 可计价，只展示计费口径 native） | `'standard'` / `'computed-only'`（日粒度 token 可计价） |
| event_charges | — | `kind='premium-requests'`（quantity=`requests.cost`，unit='requests'）；compaction 事件 `kind='ai-credits'`（quantity=`totalNanoAiu/1e9`，unit='credits'） | — | `kind='ai-credits'`（quantity=`ai_credits_used`，unit='credits'，日粒度） |
| toolCall | 不可得 | 不可得 | — | — |

### token 语义裁决（按通道 × 格式版本固定，二轮审查 §1 矩阵）

Copilot 两通道的 **cache write 与 input 的互斥性、reasoning 与 output 的包含关系**此前未经验证（只减 cache read 的旧写法被二轮判失败）。裁决规则：

1. **格式版本判别**：OTEL 通道按导出器格式版本（hrTime 形态 / 属性键集，落 `rawSchema`）；session-state 通道按 `session.start` 的 `copilotVersion` 分段。
2. **版本表固定语义**：adapter 内置版本表逐格固定 `(通道, 格式版本) × (cache write 是否 ⊂ input, reasoning 是否 ⊂ output)`；每格必须附真实样本验证证据（golden fixtures）。
3. **防御性净 input 公式**（版本表确认"cache 两档 ⊂ input"的版本）：与 Codex 同款顺序 clamp——`cacheRead = clamp(cache_read, 0, input)`；`cacheWrite5m = clamp(cache_write, 0, input − cacheRead)`；`净 input = input − cacheRead − cacheWrite5m`。确认"互斥"的版本则直接取值不 clamp。
4. **fail-visible**：版本表未覆盖的格式版本、或样本与版本表矛盾 → 事件 `status='excluded'` + `excludeReason='unresolved-token-semantics'`，记 `conflict_observations`，`rawUsage` 留存。**未经验证前不得假设任何包含关系。** 排除行 `tokenQuality` 仍为 `'native'`（excluded 是状态不是数据质量，2026-08-04 裁决补句；unverified-schema 排除行同）。

## 增量摄取策略

- **OTEL / events.jsonl**：水位线 = `(文件路径, 字节偏移)`（JSONL 类，ADR-0002），按 mtime 排序扫描目录。`COPILOT_OTEL_FILE_EXPORTER_PATH` 常见按时间戳一会话一文件，目录扫描须能发现新文件；文件大小回缩或 inode 变化视为轮转/截断，`generation++` 重扫（去重键幂等兜底）。
- **session-state 特殊处理**：`session.shutdown` 是**累计快照**，不可把多条 shutdown 的 token 数相加。只取每个 `(sessionId, model)` 的最后一条 shutdown 值（去重键天然幂等：重扫同文件重复 upsert 同键）。未 shutdown 的进行中会话只有 OTEL 可见（若开启）。
- **API（C/D 通道）：staging 整周期事务发布（二轮审查 §2）**：每个周期（月/日）先全量拉取写入 **staging**（全部 usageItems / NDJSON 行）；页序从 0 连续，且 adapter 收到 provider 的终页/完整下载集合后显式封口，才可把该周期全部页（C 的分页参数组合、D 的 download_links 全部文件）以**单事务 reconcile 发布**进 `usage_events`；**任何一页失败、缺页或未封口的成功前缀 → 整个周期不发布**，保留上一周期 projection，下轮重试。**近期周期滚动重拉**（当月 + 上一月；企业报告最近 3 天，覆盖 T+1 延迟），历史周期拉成功后封存。
- **累计值差分**：仅企业报告的 `token_usage.*_sum` 是日粒度快照值，直接当当日增量用，无需差分。

## 去重与已知坑

- **同一请求四类 OTEL 记录重复**：chat span / inference log / agent turn log / agent summary span 可能同时携带同一请求的 usage，必须按 ccusage 的优先级抑制（`should_emit_candidate`：chat > inference > agent-turn > agent-summary，按 traceId + `gen_ai.response.id` 匹配丢弃低优先级）。这是 ccusage issue #19/#313 同款教训（重复 message/sidechain 重复计费）的 Copilot 版本。
- **input_tokens 与 cache 的包含关系**：见「token 语义裁决」节——净 input 须按版本表判定后顺序 clamp 双减（cache read 与 cache write 都减），不做减法或只减 read 都会虚高。
- **OTEL 默认关闭**：绝大多数机器 `~/.copilot/otel/` 不存在（ccusage #1169 用户报"工具声称支持 Copilot 但无数据"）。Tokly 必须优先吃 session-state，OTEL 作为补充。**Tokly 红线：不得替用户开启遥测，文档只说明用户可自行开启。**
- **双源共存去重**（ccusage #1174 开放问题）：同一 session 同时有 OTEL 行与 shutdown 快照时，以 shutdown 为权威，丢弃 `gen_ai.conversation.id == sessionId` 的 OTEL 行；否则 token 双计。这是下条「四通道权威裁决」在双通道场景的特例。
- **四通道权威裁决（2026-08-02 审查 §三 P1-6 / §四 盲区 4）**：OTEL、session-state、个人 API、企业报告四通道可能同源并存，直接并库必然双计。规则：**由用户配置唯一权威通道**（设置项，默认 `session-state`——本地、默认存在、粒度最细）；非权威通道照常入库（`channel` 列区分）但 `status='excluded'`、`excludeReason='duplicate-channel'`，默认不计入任何聚合。数据保留可切换余地，双计从配置上根除。
- **shutdown 快照的 resume 稳健写法（审查遗留项）**：`--resume` 后续写的 shutdown 快照其累计语义未验证。稳健写法——新快照值 **≥ 旧值则按 upsert 覆盖**（同一会话继续累计）；**下降则另起事件**（视为新计量周期，eventKey 追加周期序号后缀），不得静默覆盖导致总量回退。
- **shutdown 时间戳 ≠ 用量时间**：会话跨天时全部 token 记在 shutdown 日，日粒度报表会出现尖峰。可在 UI 标注为"会话结算点"。
- **模型名归一化**：`claude-opus-4.7-1m-internal` 这类内部名查不到 LiteLLM 价格，需映射表；查不到时 `costComputedMicroUSD` 置 null（`computedCostStatus='unmatched'`）而非 0。
- **口径不可混算**：premium-request 数（0.04 USD/req 量级，模型有乘数）、AI Credit（1 credit = 0.01 USD）、API 等价 USD（LiteLLM）是三种不同"成本"。2026-06-01 起 premium requests 被 AI Credits 取代。数量口径一律进 `event_charges`（kind + unit 区分），USD 金额才进成本列并按 `nativeCostKind` 标注，禁止把 `requests.cost` 直接当 USD。
- **时间戳多形态**：OTEL 导出器版本差异导致 `[sec,nanos]` 数组、纳秒/微秒/毫秒/秒标量并存，按数量级判断（≥1e17 纳秒，≥1e14 微秒，≥1e11 毫秒，否则秒）。
- **企业报告限制**：下载链接限时过期（staging 流程须先下完全部文件再发布）；user 级端点不含组织代管许可的用量；报告字段可能随版本增删（官方注明 schema 为示例性质）。

## 已验证格式版本集合（2026-08-03）

- **当前为空集**：本文档撰写时调研机器无真实 Copilot CLI 会话（见文首「本机验证说明」），四通道均未拿真实样本验证——OTEL / session-state 的格式版本表（见「token 语义裁决」节）尚无一格附真实样本证据；个人 API / 企业报告的字段结构取自官方 OpenAPI 描述，同样未经真实响应验证。**M0.5 冻结门前，四通道的已验证 `raw_schema` 版本集合均为空。**
- **adapter 开工时的强制动作**：每确认一个真实样本（OTEL 导出器格式版本、`copilotVersion` 分段、或某次真实 API 响应），在此追加一行记录版本标识与验证日期，并同批补 golden fixtures——与「token 语义裁决」版本表同步冻结（ADR-0014）。
- **集合外版本的处置（ADR-0014）**：版本表补齐前，任何进入摄取管线的真实数据一律按集合外处置——事件按可解析部分入库、`status='excluded'`、`exclude_reason='unverified-schema'`，并记 `ingest_errors`（错误码 `unverified-schema`）留证；不得凭 ccusage 源码或官方文档字段描述直接判定为"已验证"。

## agent 归一化规则（ADR-0018，2026-08-03）

Copilot 四通道均无智能体/子代理（sub-agent）概念：OTEL 记录中的 "agent turn log"（`event.name == "copilot_chat.agent.turn"`）与 "agent summary span"（`gen_ai.operation.name == "invoke_agent"`）指的是同一请求的**记录类型**（见「格式详解 A」四类重复记录去重规则），不是可选择、可归一化的子代理身份，不构成 ADR-0018 定义的智能体维度。四通道 `agent` 恒 `''`。

## 能力声明

| 能力 | CLI OTEL | CLI session-state | 个人 API | 企业报告 |
|---|---|---|---|---|
| 五类 token | ✅（经版本表裁决） | ✅（经版本表裁决） | ❌ | 部分（仅 prompt/output 日总和） |
| 会话归属 | ✅ | ✅ | ❌ | ❌ |
| 项目归属 | ❌ | ✅（cwd/repo） | ❌ | ❌ |
| 智能体维度（agent） | 恒 `''` | 恒 `''` | 恒 `''` | 恒 `''` |
| 事件级时间戳 | ✅ | ❌（仅结算点） | ❌（月/日粒度） | ❌（日粒度） |
| 成本 | computed（LiteLLM） | computed + 计费数量（premium-req 数 / nano-AIU，进 event_charges） | **native**（计费口径 USD，`nativeCostKind` 标注） | ❌（`ai_credits_used` 进 event_charges） |
| 历史深度 | 文件留存期 | 文件留存期 | 24 个月 | 1 年（且自 2025-10-10 起） |
| 默认可用 | ❌ 需用户开启 | ✅ CLI 默认写入 | ✅ 需 PAT | 需企业策略+权限 |
| toolCall | ❌ | ❌ | ❌ | ❌ |

## 参考链接

- [GitHub REST API — Billing usage（premium_request/ai_credit usage 端点）](https://docs.github.com/en/rest/billing/usage?apiVersion=2022-11-28)
- [GitHub REST API — Copilot usage metrics reports](https://docs.github.com/en/rest/copilot/copilot-usage?apiVersion=2022-11-28)
- [GitHub OpenAPI 描述（github/rest-api-description）](https://github.com/github/rest-api-description)
- [Copilot usage metrics 示例 schema（github/docs）](https://github.com/github/docs/blob/main/content/copilot/reference/copilot-usage-metrics/example-schema.md)
- [ccusage Copilot 数据源文档](https://ccusage.com/guide/copilot/)（含 OTEL 环境变量说明）
- [ccusage copilot adapter 源码（parser.rs/paths.rs，OTEL 字段与去重权威实现）](https://github.com/ccusage/ccusage/tree/main/rust/adapters/copilot)
- [ccusage issue #1174（events.jsonl 读取提案，session.shutdown/compaction_complete 字段与 nano-AIU 口径）](https://github.com/ccusage/ccusage/issues/1174)
- [ccusage issue #1169（OTEL 默认关闭导致无数据）](https://github.com/ccusage/ccusage/issues/1169)

