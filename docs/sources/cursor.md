# Cursor

Cursor（VS Code 内核的 AI IDE）的会话数据落在 VS Code 状态 SQLite（`state.vscdb`）的 KV 表中，无任何官方 schema 文档且随版本漂移。**数据质量评级：估算级**——会话/消息/时间戳丰富，但 Cursor 3.x 起本地不再持久化真实 token 计数（`tokenCount` 恒为 0，本机与社区 issue 双重验证），token 只能按字符数 ÷4 估算，全程标注 `tokenQuality='estimated'`。

## 数据位置

| 平台 | 主库（global state） |
| --- | --- |
| Windows | `%APPDATA%\Cursor\User\globalStorage\state.vscdb` |
| macOS | `~/Library/Application Support/Cursor/User/globalStorage/state.vscdb` |
| Linux | `~/.config/Cursor/User/globalStorage/state.vscdb` |

辅助位置（均为 SQLite/JSON，同目录树下）：

- `User/workspaceStorage/<md5>/state.vscdb`：每工作区一个；`ItemTable` 键 `composer.composerData` 存该工作区 composer 列表（旧版本的主索引，新版本已迁移到全局库 `composerHeaders` 表）。同目录 `workspace.json` 给出 `folder` → 项目路径映射。
- `User/globalStorage/conversation-search.db`：会话标题搜索索引（`conversations` 表，含 `source/scope/id/title/updated_at`），仅可作会话发现旁证，无 token 数据。
- `~/.cursor/projects/<project>/agent-transcripts/*.jsonl`：类 Claude Code 的转写层（部分会话有），可作内容交叉校验，但 ID 体系与 composer 不完全相通。
- `~/.cursor/chats/*/*/store.db`：新版 Agent 会话的另一套存储栈（`meta`+`blobs` 表），本机未见，暂不作主源。

**读取纪律（红线，二轮审查 §2 修订）**：Cursor 运行时主库带活动 WAL 且可能加密重写，绝不直连活动库查询，也**不用裸 `cp` 复制**（活动 WAL 下可能拷到撕裂的中间态）。正确流程：以 `mode=ro` + `busy_timeout` 打开活库，用 **SQLite backup API** 把库备份到临时目录副本（一致快照，含 `-wal` 中未 checkpoint 的数据），再以 `sqlite3 "file:<副本>?immutable=1"` 只读查询副本。immutable 仅对 backup API 产出的静态副本安全；容忍分钟级滞后。

## 格式详解

主库三张表（本机 Cursor 3.x 实测）：

```sql
CREATE TABLE ItemTable (key TEXT UNIQUE ON CONFLICT REPLACE, value BLOB);
CREATE TABLE cursorDiskKV (key TEXT UNIQUE ON CONFLICT REPLACE, value BLOB);
CREATE TABLE composerHeaders (composerId TEXT PRIMARY KEY, workspaceId TEXT,
  createdAt INTEGER, lastUpdatedAt INTEGER, isArchived INTEGER,
  isSubagent INTEGER, recency INTEGER, checkpointAt INTEGER, value TEXT);
```

`cursorDiskKV` 键族（值为 JSON 文本或二进制 blob，本机分布）：

| 键模式 | 数量级 | 内容 |
| --- | --- | --- |
| `agentKv:blob:<hex-hash>` | ~14k | 请求级消息 blob，内容寻址（键是内容哈希，**无法直接 join 回会话**），JSON 者形如 `{"role":"...","content":...}`，部分为二进制/加密 |
| `bubbleId:<composerId>:<bubbleId>` | ~8k | **逐消息记录，主数据源** |
| `composerData:<composerId>` | ~130 | 会话级元数据 + `conversationMap`/`fullConversationHeadersOnly` 气泡顺序 |
| `checkpointId:<composerId>:<checkpointId>` | 少量 | 文件检查点/inline-diff 状态，与用量无关 |

bubble（每条消息）关键字段：

- `type`：`1`=user，`2`=assistant（本机 assistant 占绝大多数，user 气泡似乎会被裁剪）。
- `createdAt`：ISO-8601 文本（如 `"2026-05-14T09:32:11.482Z"`），消息时间戳。
- `text` / `richText`：消息正文。注意本机 assistant 气泡 `text` 平均仅 ~86 字符、`richText` 全空——**正文大头在 agentKv blob 里，bubble.text 严重低估输出长度**。
- `tokenCount.inputTokens` / `tokenCount.outputTokens`：**token 字段的确切位置**。老版本（≤2.x）部分会话有真实快照（社区样本 `{"inputTokens":41263,"outputTokens":4901}`）；Cursor 3.x 全部恒为 0（本机 7842 条 bubble max=0 实测，[codeburn#114](https://github.com/getagentseal/codeburn/issues/114) 同证）。取值规则：非零则 native（`tokenQuality='native'`），为零/缺失则估算（`tokenQuality='estimated'`）。
- `modelInfo.modelName`：bubble 级模型字段，但本机 7665 条 assistant 气泡**全部为 null**——模型归属只能退到会话级 `composerData.modelConfig.modelName`（另有 `maxMode`/`selectedModels`；`composerHeaders.value.forceMode/unifiedMode` 为 auto 时模型根本不可知）。
- `requestId`、`toolFormerData`/`toolResults`（工具调用名、参数、用户批准/拒绝结果）、`allThinkingBlocks`（thinking 文本，估算时计入 output 池）、`isRefunded`。

会话归属：`composerId` 在 bubble 键的第二段；`composerHeaders` 表给会话级 `createdAt`（ms epoch）、`isSubagent`、`value.workspaceIdentifier/name/totalLinesAdded`。父子关系：父会话 `composerData.subagentComposerIds`/`subComposerIds` 列出子会话，子会话侧 `composerHeaders.isSubagent=1`（本机 131 会话中 104 是子代理——**子代理流量是主体，不可忽略也不可双计**）。

原生成本：**无**。本地无任何美元金额字段（`composerData.usageData` 存在但实测恒为空对象）。Cursor 计费在云端（订阅+按量），本地只能估算。

虚构示例（结构示意，值均编造）：

```jsonc
// cursorDiskKV["bubbleId:3f9c…-aaaa:7b21…-bbbb"]
{ "type": 2, "createdAt": "2026-06-01T12:00:03.911Z",
  "text": "我先看一下这个文件。", "tokenCount": {"inputTokens": 0, "outputTokens": 0},
  "modelInfo": {"modelName": null}, "requestId": "req_9e2f…", "isRefunded": false }
```

## 归一化映射（→ Tokly UsageEvent）

| UsageEvent | 取值 |
| --- | --- |
| `source` | `"cursor"` |
| `sourceInstanceId` | 每个 `state.vscdb` 主库一实例（`dataRoot` = 库文件规范化路径） |
| `eventKey`（去重键） | **全键 `bubbleId:<composerId>:<bubbleId>`（单值定契约，2026-08-04 阶段三B 复审裁决）**——`bubbleId` 的全局唯一性是上游未承诺的观察，一旦被推翻即静默跨会话撞键（事件身份不可逆）；全键携带 composer 作用域，代价只是键长。原文"裸 ID / 稳妥写法用全键"的双读法已删——**映射表任何一格出现两个可选值即契约缺陷**，spec 是唯一施工权威，不许让 fixtures 代做设计决定 |
| `revision` | `originStream` = 库内表名 `'cursorDiskKV'`；`streamGeneration` = **DB generation**（见增量摄取节）；`sourcePosition` = 行 rowid。**rowid 只在同 generation 内可比**（ADR-0002） |
| `payloadHash` | bubble value JSON 的 sha256 |
| `sessionKey` | 键第二段 `composerId` |
| `parentSessionKey` | `isSubagent=1` 时，由父 `composerData.subagentComposerIds` 反查父 composerId；查不到则 null |
| `projectKey` / `projectLabel` | label = `composerHeaders.value.workspaceIdentifier`；缺失时经 `workspaceStorage/*/workspace.json` 的 folder 反查；再缺省 null；key 由 core 统一归一化 |
| `timestamp` | `bubble.createdAt`（ISO 文本）解析为 epoch_ms；缺失退 `composerHeaders.createdAt` |
| `model` | `bubble.modelInfo.modelName` → `composerData.modelConfig.modelName` → `null`（auto 模式；ADR-0002 禁魔法值） |
| `agent` | 恒 `''`（无智能体概念；子代理关系已由 `parentSessionKey` 表达，不重复进 agent 维度。细则见「agent 归一化规则」节，ADR-0018） |
| `tokens.input` / `tokens.output` | **字符池按 role 分离（二轮审查 §1 矩阵，禁止同内容双算输入输出）**：`tokenCount` 非零 → native 直取。为零/缺失 → 估算 `ceil(chars/4)`，且**每条 bubble 只贡献一个方向的字符池**：user 气泡（`type=1`）`text` → **input 池**；assistant 气泡（`type=2`）`text` + `allThinkingBlocks` + `toolResults` 文本总长 → **output 池**。assistant 的真实输入（上下文）本地不可得，input 只反映 user 可见输入；同一字符永不双向折算 |
| `tokens.cacheRead` / `cacheWrite5m` / `cacheWrite1h` / `reasoning` | 本地无此粒度，**一律写 0 并用 `tokenCoverage` 标注不可得**（`['cache-read','cache-write','reasoning']`；估算行另加 `'input-low-coverage'`）——0 不冒充"已确认没有"（二轮审查 §1） |
| `tokenQuality` | `tokenCount` 非零行 `'native'`（仅老版本历史数据）；估算行 `'estimated'` |
| 成本列 | `costNativeMicroUSD = null`；估算行 `pricingEligibility='disabled'`（用户显式开启「估算计价」时改 `'estimated-tokens'`）、`displayPolicy='estimate-only'`；native 历史行 `'standard'` / `'computed-only'` |
| `toolCall` | `toolFormerData.name`/`toolResults` 提取工具名与参数摘要，落 `tool_calls` 子表（以事件身份关联，整集合替换语义，ADR-0002）；无工具调用时不产生子表行 |

## 增量摄取策略

- **水位线：按隐式 rowid，且只在同 DB generation 内有效（二轮审查 §2 修订）。** `cursorDiskKV` 无自增主键，但 SQLite 隐式 rowid 单调；且 `ON CONFLICT REPLACE` 是 delete+insert——**流式生成中同一 bubble 被反复重写，每次重写都获得新 rowid**。记 `SELECT MAX(rowid) FROM cursorDiskKV` 为水位线，增量查 `WHERE rowid > :wm AND (key LIKE 'bubbleId:%' OR key LIKE 'composerData:%')`，同一 eventKey 多次出现取 rowid 最大者（upsert 语义）。
- **DB generation 检测（替代被证伪的「rowid→key 单锚点反查」）**：REPLACE 会把锚点 key 移到新 rowid，单锚点反查会误报；VACUUM、Cursor 版本迁移（如 3.0→2.6 的格式转换脚本会重写整表）或 DB 重建会使 rowid 水位线整体失效。generation 指纹 = **DB 头 file change counter + schema cookie + 首 N 行 (rowid,key) 采样哈希**；每次开扫前比对，不连续 → `streamGeneration++`、全量重扫（幂等 upsert 保证安全，旧代低 rowid 不与新代比较）。
- **轮内/跨轮分离**：每轮采集先用 backup API 产出新副本——**轮内分页在副本这一固定快照上严格向前**（天然满足"固定快照"要求）；**跨轮 checkpoint 用 safety horizon**（水位线回退重叠窗口，重复由 upsert 吸收）+ **周期全量 reconciliation** 兜底（ADR-0002 SQLite 活库类语义）。
- **触发**：轮询 `state.vscdb` 与 `state.vscdb-wal` 的 mtime（写爆发期 debounce 2s，参照 Contrails 做法）；每次采集重新备份副本，不持有连接。
- 无文件轮转/截断问题（单库）；无累计值差分问题（每行是独立消息快照，非计数器）。

## 去重与已知坑

- **eventKey 去重是第一要务**：bubble 行被流式重写产生多版本快照，必须按 `bubbleId` 去重取最新，否则输出 token 重复计数。类比 [ccusage#19](https://github.com/ryoppippi/ccusage/issues/19)：Claude Code 分支会话复制消息导致账单虚高，教训是「同一消息 ID 全局只计一次」。Cursor 侧对应场景是 checkpoint 回滚重放与 Best-of-N（`isBestOfNSubcomposer`）——落选分支的 bubble 仍在库中，应标记 `status='excluded'` 但不计入用量。
- **子代理双计**：子代理 composer 与父 composer 各自成会话；Tokly 按 eventKey 计 token 无重复，但**会话数/会话级聚合**必须用 `parentSessionKey` 归并，否则会话量虚高（本机子代理占 79%）。
- **token=0 陷阱**：Cursor 3.x `tokenCount` 存在但恒 0，若 naive 求和会得出「用量为 0」（codeburn 正是踩此坑）。必须显式分支：0/缺失 → 估算路径，并标 `tokenQuality='estimated'`；不可得的 cache/reasoning 维度写 0 并由 `tokenCoverage` 标注——**0 不冒充"已确认没有"**。
- **估算误差边界**：chars÷4 对英文 prose ≈4 字符/token 尚可，代码 ~3–4，CJK 仅 ~1.5–2（CJK 内容低估可达 50%+）；且 assistant 正文大量存于 agentKv blob（内容寻址无法 join 回会话），bubble.text 低估真实输出。**综合误差按 ±50% 声明，只能给量级，不能给账单。**
- **schema 漂移**：无官方文档；`composerHeaders` 表、`agentKv` 均为近期版本新增；加密字段（`blobEncryptionKey`、二进制 agentKv blob）随时可能扩大。adapter 必须对未知键/缺失字段宽容（skip，不 error）。
- `type=1`（user）气泡留存不全（本机 user:assistant ≈ 1:43），输入估算覆盖率低——input token 估算比 output 更不可靠，`tokenCoverage` 的 `'input-low-coverage'` 标注必须落库，UI 如实降级。

## 已验证格式版本集合（2026-08-03）

- **已验证集合**：Cursor 3.x，`state.vscdb` 三表结构（`ItemTable` / `cursorDiskKV` / `composerHeaders`，见「格式详解」）及本文档所列 bubble / composerHeaders 字段形态。本机验证：Windows，2026-08，`state.vscdb` 231MB，backup API 副本 + `immutable=1` 只读查询；`cursorDiskKV` 22361 行（`agentKv` 14334 / `bubbleId` 7842 / `composerData` 131），`composerHeaders` 131 会话（104 为 `isSubagent`）。
- **集合外的已知结构分叉，触发 unverified-schema（ADR-0014）**：仅有 workspace 级 `composerData`、全局库缺失 `composerHeaders` 表的旧版布局（见「数据位置」"旧版本的主索引"）；以及 `~/.cursor/chats/*/store.db`（`meta`+`blobs` 表，本机未见，暂不作主源）。遇到 `composerHeaders` 表缺失或改用后者存储栈的库：事件按可解析部分入库、`status='excluded'`、`exclude_reason='unverified-schema'`，记 `ingest_errors`，不按已验证形态强行解析。
- 集合演进：每验证一个新版本/存储栈，本节与 golden fixtures 同批更新（ADR-0014）。

## agent 归一化规则（ADR-0018，2026-08-03）

Cursor 无智能体/模式概念：bubble 与 composerData 均无类似 OpenCode `agent`/`mode` 或 Claude Code 子代理 `agentType` 的命名智能体标识；唯一相关信号 `composerHeaders.isSubagent`（布尔）已由 `sessionKey`/`parentSessionKey`（`composerId`/`subagentComposerIds`）表达会话层级关系，不重复进 `agent` 维度。

| 事件来源 | `agent` 取值 |
| --- | --- |
| 全部事件 | 恒 `''`（无智能体概念，ADR-0018） |

## 能力声明

| 字段 | 能力 |
| --- | --- |
| 会话/消息列表、顺序、时间戳 | ✅ 完整（composerData+bubbleId+composerHeaders） |
| 项目归属 | ✅ 大多可得（workspaceIdentifier / workspace.json 反查） |
| 模型 | ⚠️ 会话级可得，bubble 级常 null，auto 模式下不可知 |
| input/output tokens | ⚠️ 老版本部分 native；Cursor 3.x 全部为估算（chars/4，±50%，role 分离字符池） |
| cacheRead/cacheWrite/reasoning | ❌ 无（写 0 + tokenCoverage 标注） |
| 工具调用 | ✅ bubble 内 toolFormerData/toolResults（含批准/拒绝结果） |
| 智能体维度（agent） | ❌ 恒 `''`（无智能体概念，子代理关系由 `parentSessionKey` 表达，见「agent 归一化规则」节） |
| 成本 | ❌ 无原生；默认 `costNativeMicroUSD=null` 且 `pricingEligibility='disabled'`，仅在用户显式开启「估算计价」时按公开单价×估算 tokens 计算并在 UI 显著标注估算（`displayPolicy='estimate-only'`） |
| 历史深度 | 未见 TTL 清理证据；本机样本覆盖约 3 个月、131 会话/7.8k 消息，社区报告可回溯一年以上 |

一句话给 UI：**此源数字一律渲染为估算样式（`~` 前缀），永不与 native 源混排求「精确合计」。**

## 参考链接

- [vibe-replay: What Does Cursor Store on Your Machine（state.vscdb 深度逆向）](https://vibe-replay.com/blog/cursor-local-storage/)
- [codeburn#114: Cursor provider returns zero usage despite populated state.vscdb（3.x token 归零实证）](https://github.com/getagentseal/codeburn/issues/114)
- [ccusage#19: 分支会话重复计费的去重教训](https://github.com/ryoppippi/ccusage/issues/19)
- [掘金：如何解析 5 种完全不同格式的 AI 对话（Cursor 双库结构）](https://juejin.cn/post/7645553934196539442)
- [Cursor 论坛：3.0→2.6 数据格式迁移脚本（composerHeaders/conversationMap 结构佐证）](https://forum.cursor.com/t/reinstalled-cursor-2-6-from-3-and-all-my-agent-chat-history-is-gone/157014)
- [Contrails: fsnotify+debounce 监听 state.vscdb 的工程实践](https://github.com/ThreePalmTrees/Contrails)

> 本机验证：Windows，Cursor 3.x，`state.vscdb` 231MB，backup API 副本+`immutable=1` 只读查询（2026-08）。实测 cursorDiskKV 22361 行（agentKv 14334 / bubbleId 7842 / composerData 131），`tokenCount` 全 0、`modelInfo.modelName` 全 null、`composerHeaders` 131 会话中 104 为 isSubagent。示例值均为虚构。

