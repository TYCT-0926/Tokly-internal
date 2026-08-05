# OpenCode

本地优先的开源 AI 编程 Agent（sst/opencode，后迁移至 anomalyco/opencode），**v1.2.0**（2026-02-14）起从 JSON 文件存储迁移为单一 SQLite 库（版本号经 2026-08-02 审查修正：迁移发生在 v1.2.0 而非 v1.14，见 v1.2.0 release notes），含 provider 回报的五维 token 与工具自算成本。**数据质量评级：丰富级**。

> 本文 schema 经本机只读验证（验证时用 `sqlite3 "file:...?immutable=1"`；**摄取一律 `mode=ro`**，理由见「增量摄取策略」红线节；库 1.1GB / 2281 会话 / 48735 消息 / 199484 part；opencode v1.4.3），并交叉验证上游源码与 issue。示例值均为虚构。

## 数据位置

| 平台 | 路径 |
|---|---|
| Windows | `C:\Users\<user>\.local\share\opencode\opencode.db`（本机实测；全平台统一 XDG 风格，**不是** `%APPDATA%`） |
| macOS / Linux | `~/.local/share/opencode/opencode.db` |
| 附加库 | 同目录 DB 按渠道分裂为 `opencode-<channel>.db`（stable/insiders 等各自成库，schema 代际可能不同，应一并枚举）；`OPENCODE_DB` 环境变量可覆盖库路径（只读探测，不得设置） |
| 旧版 JSON（< v1.2.0，或迁移失败/被移除后遗留） | `~/.local/share/opencode/storage/{session,message,part}/` |

- 发现逻辑：先查 `OPENCODE_DB` 覆盖，再枚举 `<dataDir>/opencode*.db`；若 `storage/session/` 存在则准备 JSON 回退路径（自动迁移已移除，见「去重与已知坑」）。
- **稳定 source identity（二轮审查 §2）**：每个 `opencode*.db` 文件是一个独立 `source_instance`（`dataRoot` = 库文件规范化路径，`format_version` = Gen A/B，见下）；JSON 回退根 `storage/` 是另一个实例。实例分离后 `eventKey` 用裸 `message.id` 即可，**不再需要库文件指纹前缀**（旧表述作废）；JSON 与 SQLite 两侧出现的同 id 消息按 ADR-0002 跨流同键规则处理（payloadHash 相同去重、不同进 `conflict_observations`）。
- 库为 WAL 模式（`-shm`/`-wal` 伴随文件）。`auth.json` 在同目录，**严禁读取**（凭据）。
- 本机实测：`storage/` 下 JSON 会话目录已被迁移清理，仅剩 `migration/`、`session_diff/` 等辅助目录。

## 格式详解

### Schema 存在两代（以 `__drizzle_migrations` 最新迁移名判定）

- **Gen A**（本机主库实测，最新迁移 `20260323234822_events`）：`session` 表**无**用量列，用量只存在于 `message.data` / `part.data` 的 JSON 内。
- **Gen B**（迁移 ≥ `20260510033149_session_usage`，本机附加库实测）：`session` 表追加 `path`、`agent`、`model`（JSON text）、`cost` REAL、`tokens_input/output/reasoning/cache_read/cache_write` INTEGER（均 DEFAULT 0 的会话级累计 rollup）；新增 `session_message` 表（type 仅见 `agent-switched`/`model-switched`，**非用量数据**）。

### 两代共有的核心表

```
session(id 'ses_*' PK, project_id→project.id, parent_id, slug, directory, title, version,
        share_url, summary_additions/deletions/files/diffs, revert, permission,
        time_created, time_updated, time_compacting, time_archived, workspace_id)
message(id 'msg_*' PK, session_id→session.id, time_created, time_updated, data TEXT JSON)
part(id 'prt_*' PK, message_id→message.id, session_id, time_created, time_updated, data TEXT JSON)
project(id, worktree, vcs, name, time_created/updated/initialized, ...)
event / event_sequence          -- 事件溯源新机制，本机 0 行，暂不依赖
```

索引：`message(session_id,time_created,id)`、`part(message_id,id)`、`session(project_id/parent_id)`。全局按 `time_created` 无索引。时间戳均为**毫秒 epoch**。

### message.data JSON（token/成本的确切位置）

- **assistant 消息**（本机 42914 条 100% 含 cost+tokens）：
  ```json
  {
    "role": "assistant", "parentID": "msg_9f8e7d6c5b4a3928ZYXWVUTSRQ",
    "mode": "build", "agent": "build",
    "path": { "cwd": "D:/work/demo-project", "root": "D:/work/demo-project" },
    "providerID": "anthropic", "modelID": "claude-sonnet-4-5",
    "cost": 0.04231,
    "tokens": { "total": 51234, "input": 12345, "output": 890,
                "reasoning": 0, "cache": { "read": 38000, "write": 1200 } },
    "time": { "created": 1772500000000, "completed": 1772500012000 },
    "finish": "end_turn", "error": null, "summary": true
  }
  ```
- **user 消息**：`{role, time:{created}, agent, model:{providerID, modelID}, summary, tools, variant}`（无 token）。

### part.data JSON（按 `data.type` 区分）

- `step-finish`：`{type, reason, tokens:{...同 message 级五维...}, cost}` —— 步骤级粒度，与 message 级**同源不同粒度**。
- `tool`：`{type, tool, callID, state:{status, title, input, output, time:{start,end}, metadata}, metadata}` —— 工具调用在此。
- `text` / `reasoning` / `step-start` / `compaction`（`{type, auto}`）等，无用量。

### 成本语义

`cost` 为 **opencode 自己按 models.dev 价目表计算后写入**（**工具自算，非 provider 账单**——映射时 `nativeCostKind='tool-computed'`、`nativeCostProvider='opencode:models.dev'`，UI 不得写"原生/官方成本"，按 `displayPolicy` 展示 native vs computed 差异，二轮审查 §1）：`Session.getUsage` 将 usage 归一化后按 input/output/cache-read/cache-write 费率、reasoning 按 output 费率计价。已知缺陷：部分版本 cache-read 未计价致低估 2–3×（#28494）；Copilot/v2 runner 路径 cost=0（#30706）；无价目的 BYOK 模型 cost=0（本机附加库实测 8 会话 tokens>0 而 cost 全 0）。

### 旧版 JSON 布局（回退路径）

`storage/session/{projectID}/{sessionID}.json`、`storage/message/{sessionID}/{messageID}.json`、`storage/part/{messageID}/{partID}.json`。message/part 文件内容与现 `data` 列同形（assistant 文件含 `tokens`/`cost`），id 即文件名，跨存储去重键一致。

## 归一化映射（→ Tokly UsageEvent）

粒度：**一条 assistant message = 一个 UsageEvent**（其 tokens 是该消息所有 step 的合计）。

| UsageEvent 字段 | 取值 |
|---|---|
| source | `"opencode"` |
| sourceInstanceId | 每个 `opencode*.db` 一实例（`format_version` = Gen A/B）；JSON 回退根另为一实例（见「数据位置」稳定 identity 条） |
| eventKey（去重键） | `message.id`（`msg_*`，实例内 PK 唯一） |
| revision | `originStream` = 库内表名 `'message'`；`streamGeneration` = DB 代际（指纹断裂 ++）；`sourcePosition` = `(time_updated, id)` 复合位置的规范文本 |
| payloadHash | `message.data` 原始 JSON 的 sha256 |
| sessionKey | `message.session_id` |
| parentSessionKey | `session.parent_id`（Task 工具子代理会话，NULL=主会话） |
| projectKey / projectLabel | label = `session.project_id → project.worktree`（或 `project.name`），兜底 `session.directory`；key 由 core 统一归一化 |
| timestamp | `message.time_created`（ms epoch）；完成时间可取 `data.time.completed` |
| durationMs | `data.time.completed - data.time.created`（可得时） |
| agent / mode | `data.agent` / `data.mode`（如 `build`）；`data.agent` 缺失/降级规则见「agent 归一化规则」节（ADR-0018） |
| model | `data.providerID + "/" + data.modelID`；Gen B 的 `session.model`（JSON `{id,providerID,variant}`）只是最后使用模型，**勿作事件级模型** |
| tokens 五维 | **经版本化 resolver 判别后映射**（见下节），禁止直接复制 |
| tokenQuality | `'native'`——resolver 命中与 fail-visible 排除行同值（excluded 是状态不是数据质量，2026-08-04 裁决补句）；排除事件不入聚合 |
| costNativeMicroUSD | `round(data.cost × 1e6)`（micro-USD；源 REAL 仅此处一次换算） |
| nativeCostKind / nativeCostProvider | `'tool-computed'` / `'opencode:models.dev'`（工具自算，非账单值，见「成本语义」） |
| pricingEligibility / displayPolicy | `'standard'` / `'both'`（native vs computed 双列展示差异，#28494 低估变差异化提示） |
| toolCall | 由该 message 的 `part.data.type='tool'` 行展开：`{name: data.tool, callID, status: data.state.status, durationMs: data.state.time.end - data.state.time.start（可得时）}`，落 `tool_calls` 子表（含 status/时长，ADR-0002）。`state.title` **不映射**（2026-08-04 裁决：tool_calls 只存指标，自由文本贴内容红线，以 ADR-0002 列集为准） |

### token 语义的版本化 resolver（二轮审查 §1/§2）

`data.tokens` 五维的**包含关系依 provider 与 schema 代际而异**（reasoning 是否计入 output、cache read/write 是否计入 input、`total` 由哪些分项构成），直接复制五维值会把错误口径永久写进仓库。resolver 按序判别：

1. **版本表命中**：已验证的 `(providerID, modelID 族, opencode 版本 / schema 代际)` 组合查 adapter 内置版本表，直接定语义（版本表是代码的一部分，每格附 golden fixture 证据）；
2. **total 方程判别**：版本表未覆盖时，用 `data.tokens.total` 反推——枚举包含关系假设（reasoning⊂output？cache⊂input？total=哪些分项之和），恰有**唯一**假设使方程成立才采纳。**唯一性定义在映射结果，不在假设标签**（2026-08-04 数据门裁决）：退化输入（如 cache 与 reasoning 全零）下多个假设同时成立但映射结果相同，即为唯一解、照常采纳；『多解』专指成立的假设之间映射结果不同；
3. **fail-visible**：无法唯一判别（total 缺失、多解、与分项矛盾）→ 事件 `status='excluded'` + `excludeReason='unresolved-token-semantics'`，`rawUsage` 与 payloadHash 留存，记 `conflict_observations`（kind=`unresolved-semantics`），UI 报警可见。**禁止默认任一包含关系。**

判别结果连同 `normalizerVersion` 入库，版本表演进后可按版本重归一化。入库口径必须满足 ADR-0002 不变量：`tokens.output` 恒含 reasoning（provider 未计入时 adapter 并入，reasoning 仍保留为信息性子集）；`tokens.cacheWrite5m` = `cache.write`（无分档 → 默认 5m 档）、`cacheWrite1h = 0`。

## 增量摄取策略

**水位线：`(time_updated, id)` 复合游标，upsert 语义**——不能用自增主键（id 是时间序 ULID 风格随机串，无自增列），也不能只看 `time_created`：assistant 消息在流式生成中被反复 UPSERT（`time_updated` 递增、tokens/cost 增长、part 追加，见 #31990 的 UPSERT 证据）。上游自身分页也用 `(time_created, id)` 复合游标（`message-v2.ts` 的 `Cursor{id,time}`）。

操作语义（二轮审查 §2：SQLite 活库类——**轮内严格分页**与**跨轮 checkpoint 回退**是两种机制，分开实现、禁止混用）：

1. **单只读事务快照 + 轮内严格分页**：每轮打开**一个**只读事务（`mode=ro` + `busy_timeout`），在其一致快照内按 `SELECT id, session_id, time_created, time_updated, data FROM message WHERE (time_updated > :t) OR (time_updated = :t AND id > :id) ORDER BY time_updated, id LIMIT :page` 严格向前分页；轮内 cursor **不回退**。
2. **pending_messages 状态机（中断轮次的真实修法，替代旧"过滤后继续推进水位线"）**：只取 `data.role='assistant'` 者——`data.finish` 非空 → 定稿入库；`data.error` 非空 → 定稿入库（**error 但已有 token 的轮次照常计数**）；**15 分钟**无更新 → 陈旧定稿入库；其余按 `message.id` 登记 `pending_messages` 表，持久化其 SQLite position 与最后一次归一化 UsageEvent（只含用量字段与元数据，禁止对话内容），**每轮先按 id 重读**这些未决消息。若按 id 重读时源行已消失，立即从持久 payload 陈旧定稿并记 `pending-vanished`；不得因源行缺席丢掉已观测 token。**水位线不得越过任何 pending id**——过滤未定稿行后照样推进水位线 = 永久丢轮次（旧方案被二轮证伪）。定稿事件落库后即从 `pending_messages` 删除。
3. 定稿事件以 `message.id` upsert（同 id 覆盖，天然幂等）；水位线推进到本轮已提交事件的最大 `(time_updated, id)`，与事件写入同一事务（ADR-0002 事务边界）。
4. **跨轮 checkpoint ≠ 轮内 cursor**：跨轮用 safety horizon（checkpoint 回退 2s 重叠扫描，重复行由 upsert 吸收）+ **周期全量 reconciliation**（对账窗口内全量重扫）兜底。
5. part 表同理按 `(time_updated, id)` 增量，仅在需要 toolCall 明细时拉取。
6. 大库优化：本机 message 仅 4.8 万行，全表扫可接受；更大库可先按 `session.time_updated` 找活跃会话，再走 `message(session_id,time_created,id)` 索引。
7. 无文件轮转；但需绑定 db 文件指纹（路径+大小+mtime+DB 头 change counter），指纹剧变（删库重建、JSON→SQLite 迁移）→ `streamGeneration++`，水位线重置全量重扫。
8. **Gen B 的 `session.tokens_*`/`cost` 是累计 rollup，与 message 级求和双计**——只可用作校验，不可作事件源。

**1.1GB 大库安全读取（红线）**：

- URI 默认 `mode=ro` + `busy_timeout`：`sqlite3 "file:C:/Users/<u>/.local/share/opencode/opencode.db?mode=ro"`（只读共享锁，不与运行中的实例争锁，WAL 下读到一致快照）。Windows 版 sqlite3 CLI 需 `file:C:/...` 盘符形式，`file:/c/...` 打不开。
- **不用 `immutable=1` 读活库**（2026-08-02 审查 §四 盲区 9）：immutable 让 SQLite 假设文件不会被修改，活库 WAL 下可能读到**撕裂页**；且会忽略 `-wal` 中未 checkpoint 的尾部数据。immutable 仅对静态副本安全（如 Cursor 经 backup API 产出的副本，见 docs/sources/cursor.md）。
- 绝不执行任何 `PRAGMA` 写操作（如 `wal_checkpoint`、`journal_mode`）。
- 查询一律分页 LIMIT，只 SELECT 需要列，避免 `SELECT *` 拉大 JSON。

## 去重与已知坑

- **去重键 = `message.id`**（实例内 PK 保证唯一）。多 db 文件（`opencode-*.db`）分属不同 `source_instance`，不存在跨实例撞键问题；JSON 回退路径与 SQLite 库出现同 id 消息时按跨流同键规则：payloadHash 相同去重、不同进 `conflict_observations`。
- **子代理不是重复**：Task 工具生成 `parent_id` 非空的子会话，其 assistant 消息是真实 token 消耗，必须计入，经 `parentSessionKey` 归属（参考 ccusage #19 对子代理计费的争论、#313 重复事件教训）。
- **step-finish part 与 assistant message 的 tokens/cost 是同一份数据的两种粒度**，二选一，混用双计；Gen B session rollup 同理（三选一）。
- **compaction**：压缩后旧消息仍留库（`filterCompacted` 仅运行时过滤），按 msg_id 摄取不受影响；session 有 `time_compacting` 标记可供展示。
- **revert/分支**：`session.revert` 与被回滚消息的清理行为未完全验证；摄取按库内现状全量计入， Tokly 若需"有效用量"应另做过滤层。
- **版本差异**：v1.2.0 前为 JSON 存储（ccusage #966：只找 JSON 目录会误报"无数据"）；JSON→SQLite 自动迁移**后被移除**（anomalyco/opencode#34445）——遗留 JSON 永远不会被自动导入；且迁移存在静默跳过 bug（#13654），老 JSON 可能孤立。`storage/session/` 存在时一律准备 JSON 回退路径（独立 source_instance），其中 db 里不存在的 ses id 走回退补采。
- **cost 低估**：#28494（cache-read 未计价）、#30706（cost=0）；无价目模型 cost=0。cost=0 且 tokens>0 的事件 `pricingEligibility` 仍为 `'standard'`——Tokly 自算列会给出 computed 值，`displayPolicy='both'` 下差异直接可见。
- `tokens.total` 可能 ≠ 分项和——这正是版本化 resolver 的判别输入，见映射节。

## 已验证格式版本集合（2026-08-03）

- **已验证集合**：SQLite Gen A（`session` 无用量列，`__drizzle_migrations` 最新迁移 `20260323234822_events`，本机主库只读实测）与 Gen B（迁移 ≥ `20260510033149_session_usage`，`session` 追加累计 rollup 列，本机附加库只读实测），均取自 opencode **v1.4.3**（1.1GB / 2281 会话 / 48735 消息 / 199484 part）。
- **旧版 JSON 布局（< v1.2.0）不在本机实测集合内**：本机 `storage/` 下会话 JSON 已被迁移清理无残留内容，该布局仅交叉验证上游源码（message-v2.ts）与 issue（#966/#34445/#13654），无本地 golden fixture 支撑。
- 与「token 语义的版本化 resolver」按 provider/model 判别 token 包含关系的版本表是正交维度：本节针对库/文件**结构**版本（`raw_schema`/`format_version`），resolver 表针对**字段语义**（reasoning/cache 归属）。
- **unverified-schema 触发条件（ADR-0014）**：`__drizzle_migrations` 出现集合外迁移名（既非 Gen A 已知迁移、也不落在 Gen B 已知区间），或 JSON 回退文件的字段结构与「旧版 JSON 布局」节描述不符——事件按可解析部分入库、`status='excluded'`、`exclude_reason='unverified-schema'`，记 `ingest_errors`（错误码 `unverified-schema`，`detail_key` 记迁移名/版本标识），不默认套用 Gen A/B 或 JSON 布局语义。
- 集合演进：每支持一个新迁移代际，本节与 golden fixtures 同批更新（ADR-0014）。

## agent 归一化规则（ADR-0018，2026-08-03）

| 事件来源 | `agent` 取值 |
|---|---|
| assistant 消息（主会话或 Task 子代理会话，同一规则统一适用——`agent` 是消息级字段，不按会话层级特判） | `data.agent` 原值（如 `build`/`plan`/自定义 agent 名） |
| `data.agent` 缺失或为空字符串 | `''`（无智能体，ADR-0018 哨兵值；**不回退到** `data.mode`——mode 是独立聚合维度，见「归一化映射」表，二者不得合并） |

## 能力声明

- 可提供：五维 token（input/output/reasoning/cacheRead/cacheWrite，经版本化 resolver）、providerID+modelID、工具自算成本、session/parentSession 归属、project 归属、工具调用明细、ms 级时间戳。
- 历史深度：全量本地历史，opencode 无自动清理机制（#22110），本机实测 1.1GB 库可全量只读扫描。
- 成本：**native（工具按 models.dev 自算，非 provider 账单）**——存在历史低估 bug；UI 按 `displayPolicy='both'` 展示 native vs computed 差异，不单独背书"原生"。

## 参考链接

- [ccusage #966 — OpenCode 改用 SQLite 后 JSON 目录失效](https://github.com/ryoppippi/ccusage/issues/966)
- [anomalyco/opencode #34445 — JSON→SQLite 自动迁移被移除，遗留 JSON 需回退路径](https://github.com/anomalyco/opencode/issues/34445)
- [anomalyco/opencode #13654 — JSON→SQLite 迁移静默跳过，旧 JSON 孤立](https://github.com/anomalyco/opencode/issues/13654)
- [anomalyco/opencode #28494 — cost 计算忽略 cache-read 计价](https://github.com/anomalyco/opencode/issues/28494)
- [anomalyco/opencode #30706 — v2 SessionRunner cost=0、Copilot 缓存计价缺陷](https://github.com/anomalyco/opencode/issues/30706)
- [anomalyco/opencode #31990 — part 表 UPSERT 证据（流式更新语义）](https://github.com/anomalyco/opencode/issues/31990)
- [anomalyco/opencode #22110 — 存储无清理机制，无限增长](https://github.com/anomalyco/opencode/issues/22110)
- [上游源码 message-v2.ts — 表结构水合与 (time,id) 游标分页](https://raw.githubusercontent.com/sst/opencode/dev/packages/opencode/src/session/message-v2.ts)
- [models.dev — opencode 成本价目来源](https://models.dev/)

