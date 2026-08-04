# Gemini CLI

本地 JSONL 追加式操作日志，逐模型响应记录五分类 token（input/output/cached/thoughts/tool）+ 模型 + 会话归属，无原生成本。数据质量评级：**丰富**（token 维度与 Claude Code 同级，但需注意 input 与 cached 的包含关系歧义，且存储目录自命名为 tmp、易失）。

## 数据位置

| 平台 | 路径 |
|---|---|
| macOS / Linux | `~/.gemini/tmp/<projectId>/chats/*.jsonl`（旧版为 `*.json` 整体 JSON） |
| Windows | `%USERPROFILE%\.gemini\tmp\<projectId>\chats\*.jsonl` |
| 自定义根目录 | 环境变量 `GEMINI_CLI_HOME` 存在时替换 `~/.gemini`（只读探测，不得设置） |

- **projectId 有两代**（本机实测两代并存）：
  - 旧版：项目根路径原始字符串的 **sha256 hex**（64 位十六进制目录名）。注意哈希输入保留原始大小写与反斜杠（本机验证：`sha256("E:\Axiom\orca")` 命中头部 projectHash，而小写形式不命中），**不可由注册表路径反推复算**。
  - 新版（约 0.8+）：`~/.gemini/projects.json` 注册表把归一化项目路径映射为人类可读 slug（如 `orca`），并在 `tmp/<slug>/.project_root` 写入属主路径标记。Windows 下注册表键与标记内容均为**小写归一化**路径。首次运行新版时自动把旧 hash 目录整体迁移为 slug 目录（目录改名事件！）。
- 项目归属解析顺序：`tmp/<dir>/.project_root` 文件内容 → `projects.json` 反查 → 目录名兜底。**禁止**尝试反解 projectHash。
- 子代理会话：`chats/<parentSessionId>/<subagentSessionId>.jsonl`，父会话 ID 即上级目录名。
- 相邻文件：`history/<projectId>/` 是文件检查点的影子 git 仓库（/rewind 用，无用量数据）；`settings.json` 含遥测配置（见下）；`oauth_creds.json`、`google_accounts.json` 为凭据，**禁止读取**。`tmp/<projectId>/logs/` 为调试日志，忽略。

## 格式详解

每个 `.jsonl` 文件是一个会话的**追加式操作日志**（非快照），每行一个 JSON。解析器必须按上游 `loadConversationRecord` 语义折叠状态，不能逐行当独立事件。

### 行类型清单（白名单：只有 gemini 消息行携带 token）

| 行形态 | 处理 | 说明 |
|---|---|---|
| `{sessionId, projectHash, startTime, lastUpdated, kind, ...}` | 读取元数据 | 首行头部；`kind` 为 `main`/`subagent`；后续同名 `$set` 可更新 |
| 消息行 `{id, timestamp, type, content, ...}` | `type=="gemini"` 且有 `tokens` 时**消费**；其余跳过 | `type ∈ user / gemini / info / error / warning`；同一 `id` 可出现多次（先无 tokens 后补 tokens），**按 id 折叠、后者覆盖前者** |
| `{"$set": {...}}` | **`$set.messages` 必须执行全量检查点**；其余 `$set` 仅维护元数据 | `$set.messages` 是上游写的全量检查点（清空重建，消息 id 不变）——跳过它会导致折叠状态与源背离（二轮审查 §1/§2）；处理规则见「增量摄取策略」第 3 条 |
| `{"$rewindTo": "<messageId>"}` | 不删用量 | 会话回滚标记；API 调用已发生、token 已消耗，用量事件保留（见去重节） |
| 旧版 `.json` 整体文件 | 兼容解析 | 单 JSON `{sessionId, ..., messages: [...]}`，字段相同；resume 时自动重写为 `.jsonl`（路径变化） |

### token 字段的确切位置（gemini 消息行）

```jsonc
// 以下为虚构示例，仅示意结构
{
  "id": "7c1e9b42-3f0a-4e55-9a21-000000000042",   // 消息 uuid：去重键
  "timestamp": "2026-07-22T09:14:05.311Z",        // ISO 8601 UTC
  "type": "gemini",
  "model": "gemini-2.5-pro",                       // 模型字段；合成消息可能缺省
  "content": [{ "text": "（虚构示例文本）" }],
  "thoughts": [{ "subject": "（虚构）", "description": "（虚构）", "timestamp": "..." }],
  "toolCalls": [{ "id": "call_abc123", "name": "read_file", "args": {}, "status": "success", "timestamp": "..." }],
  "tokens": {                 // 映射自 GenerateContentResponseUsageMetadata
    "input": 12480,           // promptTokenCount（**含 cached 部分**，见口径要点）
    "output": 312,            // candidatesTokenCount
    "cached": 10240,          // cachedContentTokenCount（input 的子集）
    "thoughts": 158,          // thoughtsTokenCount（可选）
    "tool": 96,               // toolUsePromptTokenCount（可选）
    "total": 23146            // totalTokenCount（用于包含关系判别）
  }
}
```

口径要点（ccusage Rust adapter 实测口径）：Gemini API 对 `cached`/`tool` 是否计入 `input` 的语义在版本间不一致，**用 `total` 反推**：设 `inclusive = input + output + thoughts + tool`、`exclusive = inclusive + cached`，若 `cached > 0` 且 `total == inclusive != exclusive`，则 `input` 已含 `cached` → 未缓存输入 = `input - min(input, cached)`；否则视 `input` 与 `cached` 互斥。`tool` 计入 input，`thoughts` 计价时并入 output。

**失败策略（二轮审查 §1 矩阵：异常路径必须显式）**：`total` 缺失、或 inclusive/exclusive 两种方程都不成立时，**不默认任何缓存语义**——事件 `status='excluded'` + `excludeReason='unresolved-cache-semantics'`，`rawUsage` 留存，记 `conflict_observations`（kind=`unresolved-semantics`），fail-visible；禁止硬猜互斥或包含。

**无原生成本字段**，成本必须由定价表计算（免费层 Code Assist 用户同样按表估算仅供参考）。

## 归一化映射

| UsageEvent 字段 | 取值规则 |
|---|---|
| `source` | 常量 `'gemini-cli'` |
| `sourceInstanceId` | core 按 `(source, account, machine, dataRoot)` 创建；`dataRoot` = `~/.gemini` 或 `GEMINI_CLI_HOME` 规范化路径 |
| `eventKey`（去重键） | gemini 消息行的 `id`（uuid，全局唯一且 resume/迁移重写后不变）。**禁止**用「文件路径+行号」——同一消息会先无 tokens 后有 tokens 出现两次，且 `.json`→`.jsonl` 迁移、hash→slug 目录迁移都会改路径 |
| `revision` | `originStream` = 相对 tmp 根的文件路径；`streamGeneration` = 文件身份代际；`sourcePosition` = 字节偏移（只在同 stream 内可比，ADR-0002） |
| `payloadHash` | 该行原始字节的 sha256 |
| `channel` / `status` | chats 文件通道：`channel='chats'`；被动 OTEL 通道：`channel='otel'`。两通道并存时权威通道计 `status='counted'`，另一通道入库但 `status='excluded'` + `excludeReason='duplicate-channel'`（见能力声明·OTEL 节的权威裁决） |
| `sessionKey` | 文件头部 `sessionId`（= 会话首个 prompt 的 promptId，可与 OTEL 事件 `prompt_id` 关联） |
| `parentSessionKey` | 子代理文件：上级目录名（父 sessionId）；主会话：缺省 |
| `projectKey` / `projectLabel` | label = `.project_root` 标记 / `projects.json` 反查得到的项目路径；key 由 core 统一归一化 |
| `timestamp` | 消息行 `timestamp` 解析为 epoch_ms |
| `model` | 消息行 `model`；缺省时回退本文件内上一条 gemini 消息的 model，再兜底 `null`（ADR-0002 禁魔法值） |
| `tokens.input` | 拆分后的未缓存输入 + `tool`（见口径要点与失败策略；净值，符合 ADR-0002 口径不变量） |
| `tokens.output` | `tokens.output + tokens.thoughts`——output 恒含 reasoning（口径不变量，映射时并入） |
| `tokens.cacheRead` | `tokens.cached` |
| `tokens.cacheWrite5m` / `cacheWrite1h` | 0（Gemini 无缓存写入计费维度）；`tokenCoverage` 标注 `'cache-write'` 维度不存在 |
| `tokens.reasoning` | `tokens.thoughts ?? 0`（信息性子集，已含在 output 内，不重复计价） |
| `tokenQuality` | `'native'` |
| 成本列 | `costNativeMicroUSD = null`；`pricingEligibility='standard'`；`displayPolicy='computed-only'` |
| `toolCall` | 可选：gemini 消息 `toolCalls[]` 的 `name`/`status`，落 `tool_calls` 子表（事件身份关联），不再发零 token 独立事件（ADR-0002） |

## 增量摄取策略

水位线按 ADR-0002（JSONL 类）：`(sourceInstanceId, channel, 相对路径, 'jsonl-offset')` + 字节偏移 + mtime + 文件大小。

1. 文件级**纯追加**（消息/ `$set` / `$rewindTo` 均为 append）→ 按字节偏移续读；mtime/大小未变则跳过。
2. **行尾残缺**：上游用 `appendFileSync` 逐行写，崩溃可留半行。每轮只消费到最后一个完整 `\n`，尾部解析失败不推进水位线；完整非法行记 `ingest_errors` 后推进。
3. **`$set.messages` 全量检查点必须执行（二轮审查 §2）**：遇到 `$set.messages` 时清空本流已折叠状态、按检查点内容重建（消息 id 不变）；增量侧实现为该流 `generation++` 自检查点行起全量应用（最简形态：重置水位线从头重读）。已入库事件按 `eventKey` upsert 收敛；检查点中**未出现的事件不删除**——缺席 ≠ 删除，observation 永久保留（ADR-0002 两层模型）。
4. **同 id 覆盖**：增量行与历史行撞 `eventKey` 时按「同 stream 内后者覆盖」upsert（主要体现在 tokens 后补）；幂等性由 `(sourceInstanceId, eventKey)` 主键保证。
5. **截断/重写检测**：文件大小 < 水位线偏移 → `generation++` 全量重读。`.json`→`.jsonl` 迁移与 hash→slug 目录迁移表现为旧路径消失 + 新路径出现：旧水位线静默丢弃，新文件全量读取，靠 `eventKey` 全局去重防双计。
6. **无累计差分需求**：`tokens` 是单次响应瞬时值。
7. 扫描范围：`tmp/*/chats/*.json*` 及 `tmp/*/chats/*/*.jsonl`（子代理层）两层通配；同时兼容旧 hash 目录与未迁移安装。

## 去重与已知坑

1. **消息 id 折叠是必须，不是优化**：`recordMessageTokens` 先把无 tokens 的消息行落盘，拿到 usageMetadata 后**再追加一条同 id 带 tokens 的完整消息**。逐行独立计数会把同一响应计成「0 token + N token」两条；ccusage Rust 实现同样以 id 建索引后者覆盖（`direct_event_indexes`）。
2. **分支/恢复不复制用量**：与 [ccusage issue #19](https://github.com/ryoppippi/ccusage/issues/19)（Claude Code 分支复制事件）不同，Gemini CLI resume 复用原文件、消息 id 保持不变；但**目录迁移**会造成跨路径同 id 重读，故去重必须跨文件全局生效（Tokly `(sourceInstanceId, eventKey)` 主键满足）。`$rewindTo` 回滚只影响会话视图，不回退已发生的 API 消耗——用量事件保留，不按 rewind 删除。
3. **子代理用量必须计入**（[ccusage issue #313](https://github.com/ryoppippi/ccusage/issues/313) 教训）：`kind=="subagent"` 的文件在 `chats/<parentSessionId>/` 子目录，文件名是完整子会话 id（非 `session-` 前缀格式），漏扫该层会低估。
4. **input/cached 包含关系歧义**：见格式详解口径要点与失败策略；直接 `input - cached` 在 disjoint 数据上会算成负数，直接相加又可能双计，必须先判 `total`；判不出来就进冲突表，不默认语义（二轮审查 §1）。这是 ccusage 处理 gemini 数据的核心教训。
5. **tmp 目录易失性**：目录自命名 `tmp`；无可恢复内容的会话（无用户实质输入）退出时自动删除（`deleteCurrentSessionIfNotResumableAsync`）；`/chat delete` 手动删除；磁盘满（ENOSPC）时静默停录。未发现定时清理，但 adapter 必须容忍文件随时消失，历史价值靠 Tokly 本地仓库沉淀。
6. **IDE 内嵌会话**：VS Code 伴侣（a2a-server）产出的会话 `sessionId` 为字面量 `"a2a-server"`（本机实测），同项目多文件同 sessionId——此时 eventKey（消息 uuid）仍是唯一防线，sessionKey 仅作展示分组。
7. **版本差异**：旧版整体 `.json`、新版 JSONL 操作日志、hash/slug 两代目录并存；坚持「白名单只取 `type=="gemini"` 且 `tokens` 为对象的记录 + 可选字段兜底」，不为特定版本写分支。

## 能力声明

| 能力 | 支持 | 备注 |
|---|---|---|
| 逐响应 token（input/output/cacheRead/reasoning） | ✅ | 五分类（含 tool），需 total 判别拆分 cached；判别失败进冲突表 |
| cacheWrite | ❌ | Gemini 无此维度（写 0 + tokenCoverage 标注） |
| 模型 | ✅ | 消息行 `model`，少数合成消息缺省 |
| 时间戳 / 项目 / 会话归属 | ✅ | 消息级 timestamp；项目靠 `.project_root`/`projects.json` |
| 工具调用 | ✅ | gemini 消息 `toolCalls[]` |
| 子代理用量 | ✅ | `chats/<parentSessionId>/` 子目录 |
| 成本 | computed | 无原生字段，定价表计算；免费层用户仅供参考 |
| 历史深度 | 受源限制（脆弱） | tmp 目录：无自动清理但会随手动删除/不可恢复会话清除/迁移而变动 |
| `/stats` 命令 | ❌ 不可被动利用 | 纯内存会话统计（`context.session.stats`），不落盘，进程退出即失 |
| OTEL 遥测 | ⚠️ 条件性被动利用 | 见下 |

**OTEL 被动通道**（严格只读，遵守 ADR-0003，Tokly 不得替用户开启）：若用户已自行在 `settings.json` 配置 `telemetry.enabled=true` 且设了 `telemetry.outfile`，该文件是 OTLP JSON 流，其中 `gemini_cli.api_response` 日志事件含完整 token 明细（`input_token_count`/`output_token_count`/`cached_content_token_count`/`thoughts_token_count`/`tool_token_count`/`total_token_count` + `model` + `prompt_id` + 公共属性 `session.id`），粒度与 chats 文件一致且每 API 调用一条（比消息行更细）。仅当探测到该 outfile 已存在时作为补充源只读消费；本机 settings.json 无 `telemetry` 键（未开启，符合默认）。

**权威通道与跨通道 eventKey（二轮审查 §2）**：chats 与被动 OTEL 可能同时覆盖同一批 API 调用，直接并库必然双计。规则：**权威通道 = chats**（默认存在；用户可配置切换为 OTEL，与 Copilot 四通道同一模式，ADR-0002）——权威通道事件 `status='counted'`，另一通道照常入库（`channel` 列区分）但 `status='excluded'` + `excludeReason='duplicate-channel'`。两通道**无稳定共享事件 id**（chats 用消息 uuid，OTEL 用 span 属性与 `prompt_id`），故**不做行级跨通道去重**——靠通道级整体排除防双计；两通道的 `eventKey` 各按本通道规则构造，互不混用。

## 参考链接

- [gemini-cli 源码：chatRecordingService.ts（JSONL 写入/折叠语义、tokens 映射）](https://github.com/google-gemini/gemini-cli/blob/main/packages/core/src/services/chatRecordingService.ts)
- [gemini-cli 源码：chatRecordingTypes.ts（TokensSummary/MessageRecord 定义）](https://github.com/google-gemini/gemini-cli/blob/main/packages/core/src/services/chatRecordingTypes.ts)
- [gemini-cli 源码：storage.ts / projectRegistry.ts（hash→slug 迁移、projects.json）](https://github.com/google-gemini/gemini-cli/blob/main/packages/core/src/config/storage.ts)
- [gemini-cli 官方遥测文档（telemetry.outfile、gemini_cli.api_response 字段）](https://github.com/google-gemini/gemini-cli/blob/main/docs/cli/telemetry.md)
- [ccusage Rust gemini adapter（total 判别 cached 包含关系、id 折叠参考实现）](https://github.com/ryoppippi/ccusage/tree/main/rust/adapters/gemini)
- [ccusage issue #19 — 分支会话重复计数](https://github.com/ryoppippi/ccusage/issues/19)
- [ccusage issue #313 — 子任务 token 漏计](https://github.com/ryoppippi/ccusage/issues/313)
- [LiteLLM 价格表（computed 成本来源）](https://github.com/BerriAI/litellm/blob/main/model_prices_and_context_window.json)

> 本文档格式细节经本机只读抽样验证（Windows，`~/.gemini`：4 个项目目录、7 个 chats 文件，hash 与 slug 两代布局并存，projectHash 大小写敏感已实测）+ 上游 main 分支源码核对；本机样本会话过短未含 gemini 响应行，tokens 结构以源码与 ccusage 实现双重佐证；示例均为虚构脱敏数据。
