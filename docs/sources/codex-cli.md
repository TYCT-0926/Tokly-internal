# Codex CLI

OpenAI Codex CLI 的本地会话记录（rollout JSONL），按行持久化会话元数据、对话条目与 token 用量事件。
**数据质量评级：丰富** —— 原生记录每次 API 调用的 input/cached/cache_write/output/reasoning 细分 token 数；无原生美元成本，需按价格表 computed。

## 数据位置

| 平台 | 路径 |
| --- | --- |
| macOS / Linux | `~/.codex/sessions/YYYY/MM/DD/rollout-<时间戳>-<uuid>.jsonl` |
| Windows | `%USERPROFILE%\.codex\sessions\YYYY\MM\DD\rollout-<时间戳>-<uuid>.jsonl`（即 `C:\Users\<用户>\.codex\...`） |
| 归档（全平台） | `<CODEX_HOME>/archived_sessions/YYYY/MM/DD/...`（同构） |

- 根目录可被环境变量 `CODEX_HOME` 覆盖（默认 `~/.codex`）；支持多 Codex home 时应逐个扫描（每个 home 是一个独立 `source_instance`）。
- 文件名中的 `<uuid>` 即线程 ID；`<时间戳>` 形如 `2026-01-15T08-30-00`（本地时区，建文件时刻）。
- `archived_sessions/` 是用户在 TUI `/archive` 或 app-server `thread/archive` 归档后的存放处，文件从 `sessions/` **移动**过去。两处出现同一相对路径时 `sessions/` 优先（防止双计）。归档文件格式与活跃文件完全一致，必须一并摄取。
- 本机验证（2026-08，只读）：269 个 rollout 文件，cli_version 0.133.0-alpha.1 – 0.146.0；`archived_sessions/` 为空目录但代码路径存在。

## 格式详解

每行一个 JSON 对象：`{"timestamp": "<RFC3339>", "type": "<记录类型>", "payload": {...}}`。行级 `timestamp` 为 UTC，是事件时间戳的唯一可靠来源。

### 记录类型清单（`type` 字段）

| type | 说明 | 摄取价值 |
| --- | --- | --- |
| `session_meta` | 会话元数据，正常文件仅首行；fork/子代理文件会内嵌父历史副本而出现多行 | sessionId / project / 父会话 |
| `turn_context` | 每轮上下文，`payload.model` 是**模型字段的唯一可靠来源** | 模型状态机 |
| `event_msg` | 事件，`payload.type` 区分子类型（见下） | token 用量在此 |
| `response_item` | Responses API 条目（`message` / `reasoning` / `function_call` / `function_call_output` / `custom_tool_call` 等） | 可选：toolCall 计数 |
| `compacted` | 压缩后的历史替换（`message` + `replacement_history`） | 忽略 |
| `world_state` / `inter_agent_communication_metadata` | 新版多代理机制 | 后者是子代理重放边界标记 |

`event_msg` 的 `payload.type` 实测出现：`token_count`、`task_started`、`task_complete`、`user_message`、`agent_message`、`turn_aborted`、`thread_settings_applied`、`context_compacted`、`sub_agent_activity`、`mcp_tool_call_end`、`patch_apply_end` 等。上游 `rollout/src/policy.rs` 的白名单决定哪些会落盘，随版本演进。

### token 字段的确切位置（核心）

`type="event_msg"` 且 `payload.type="token_count"`，上游定义（`protocol/src/protocol.rs`）：

```jsonc
{
  "timestamp": "2026-01-15T08:31:02.451Z",
  "type": "event_msg",
  "payload": {
    "type": "token_count",
    "info": {                       // 可能为 null（任务开始/仅刷新费率时），用量事件跳过（rate_limits 不受影响，见增量摄取节）
      "total_token_usage": {        // 自会话开始的逐次 API 调用累计值
        "input_tokens": 152300,        // 含缓存（cached 与 cache_write 都是其 details，见映射表公式）
        "cached_input_tokens": 140000, // 缓存命中（cache read，input 的 details）
        "cache_write_input_tokens": 0, // 缓存写入（cache write，同为 input 的 details）；新字段，旧版本无此键（serde default=0）
        "output_tokens": 2400,         // 含 reasoning（子集）
        "reasoning_output_tokens": 900,
        "total_tokens": 154700         // 语义=当前上下文窗口占用，不是计费口径
      },
      "last_token_usage": { ... },  // 同上结构；最近一次 API 调用的增量
      "model_context_window": 272000
    },
    "rate_limits": { "limit_id": "...", "primary": null, "...": "..." }  // 官方配额快照（非美元成本）→ quota_snapshots 表，见 ADR-0008
  }
}
```

字段语义经上游 `impl TokenUsage` 与二轮审查上游测试双重证实：**`cached_input_tokens` 与 `cache_write_input_tokens` 都是 `input_tokens` 的 details**——净输入必须两者都减（上游测试用例：input=100 / read=40 / write=60 → 净 input=0；只减 read 的旧公式得 60，双计 cache write，二轮审查 §1/§3 推翻一轮"差分全对"结论）；`output_tokens` 已含 `reasoning_output_tokens`；`total_token_usage` 由 `last_token_usage` 逐次 `add_assign` 累加。`token_count` 每次模型响应落盘一条。

### 会话/项目/模型归属字段

- `session_meta.payload`：`id`（线程 uuid，与文件名一致）、`session_id`（新版）、`forked_from_id`、`parent_thread_id`、`cwd`（项目归属）、`cli_version`、`model_provider`、`originator`、`source.subagent.thread_spawn{parent_thread_id, depth, agent_path, agent_nickname}`（子代理标识）、`git{commit_hash, branch, repository_url}`（新版）。
- `turn_context.payload`：`turn_id`、`cwd`、`model`、`effort` 等。一个文件可有多个（中途切模型），解析时维护"最近 model"状态机。
- **无原生成本字段**。`rate_limits` 只有配额百分比/credits 快照。

## 归一化映射（→ Tokly UsageEvent）

| UsageEvent 字段 | 取值 |
| --- | --- |
| `source` | `"codex"`（二轮审查 §1 交叉一致性：与核心六值枚举对齐，替代旧 spec 的 `"codex-cli"`） |
| `sourceInstanceId` | core 按 `(source, account, machine, dataRoot)` 创建 `source_instances` 行；`dataRoot` = `CODEX_HOME` 或 `~/.codex` 规范化路径；格式代际记入实例 `format_version`（`'legacy'` / `'paginated'`，见坑 7） |
| `eventKey`（去重键） | 无原生事件 ID。流内主键用 `相对文件路径:字节偏移`（摄取幂等）；跨流辅键用内容哈希 `sha256(timestamp\|model\|input\|cached\|cache_write\|output\|reasoning\|total)`——对应 ccusage 的内容键去重，可吸收 fork 复制出的重复事件。跨流同键按 ADR-0002 收敛契约：payloadHash 相同去重，不同进 `conflict_observations` |
| `revision` | `originStream` = 相对 sessions 根的路径；`streamGeneration` = 文件身份代际；`sourcePosition` = 字节偏移（paginated 布局按页内偏移，见坑 7） |
| `payloadHash` | 该行原始字节的 sha256 |
| `sessionKey` | 文件内**第一条** `session_meta.payload.id`（后续 meta 行属父历史，忽略） |
| `parentSessionKey` | `session_meta.payload.forked_from_id`，缺省取 `source.subagent.thread_spawn.parent_thread_id` |
| `projectKey` / `projectLabel` | label = `session_meta.payload.cwd`（可用 `turn_context.cwd` 兜底）；key 由 core 统一归一化 |
| `timestamp` | 行级 `timestamp`（UTC RFC3339）解析为 epoch_ms |
| `model` | 该条 token_count 之前最近的 `turn_context.payload.model`；无 turn_context 的早期文件（2025-09 上旬）置 `null`（ADR-0002 禁魔法值；由 pricing worker 落 `unmatched`，缺口可见） |
| `tokens.input` | **净输入公式（二轮审查修正）**：对差分量顺序 clamp 后同时减两种 cache——`cacheRead = clamp(cached_input_tokens, 0, input_tokens)`；`cacheWrite = clamp(cache_write_input_tokens ?? 0, 0, input_tokens − cacheRead)`；`tokens.input = input_tokens − cacheRead − cacheWrite`。cache read 与 cache write **都是** input 的 details（上游测试：100/40/60 → 0），只减 read 会虚高 |
| `tokens.cacheRead` | 上式的 `cacheRead` |
| `tokens.cacheWrite5m` | 上式的 `cacheWrite`（无 5m/1h 分档概念，全部计默认 5m 档，ADR-0002）；`tokens.cacheWrite1h = 0` |
| `tokens.output` | `output_tokens`（含 reasoning，符合口径不变量） |
| `tokens.reasoning` | `reasoning_output_tokens`（信息性，是 output 子集，不重复计价） |
| `tokenQuality` | `'native'` |
| `serviceTier` | `thread_settings_applied`（0.144.0+）的 `service_tier`（priority/fast/default），缺省 null |
| 成本列 | `costNativeMicroUSD = null`（无原生成本）；`pricingEligibility='standard'`；`displayPolicy='computed-only'` |
| `toolCall` | 可从 `response_item` 的 `function_call` / `custom_tool_call` / `local_shell_call` 提取（可选增强），落 `tool_calls` 子表（ADR-0002） |
| → `quota_snapshots` | 每条 `token_count` 的 `rate_limits` 是官方配额快照，落 `quota_snapshots` 表（`provenance='official'`），零推算——见 [ADR-0008](../adr/0008-配额窗口.md)。**采集独立于下方的 token 跳过逻辑**（增量摄取节第 6 条） |

## 增量摄取策略

- **水位线：按文件字节偏移 + 折叠快照**。rollout 文件只追加、不轮转；为每个文件持久化 `(fileId/相对路径, offset)` 水位线与**带格式版本的折叠快照** `{formatVersion, previousTotals, currentModel, replayState}`（二轮审查 §2：previous_totals 持久化二选一，采纳"持久化带格式版本的折叠快照"）。
- **水位线恢复**：快照是快路径，重放是兜底——恢复时 `formatVersion` 与当前解析器不符（格式演进）即丢弃快照，从文件头顺序重放至 offset 重建差分状态（文件通常 < 数 MB，直接顺序重放，不持久化全量中间态）。
- **差分算法**（对每条 `token_count`，与 ccusage 对齐）：
  1. `payload.info == null` → 跳过用量（**rate_limits 仍采集**，见第 6 条）；
  2. `total_token_usage == previous_totals` → 跳过（重放/重复快照，未前进）；
  3. 增量优先取 `last_token_usage`；缺失或累计未前进时退化为 `total - previous_totals`；
  4. input/cached/cache_write/output/reasoning **五项**计费字段全 0 → 跳过（同时挡掉 auto-compact 合成事件，见坑 3）；
  5. 更新 `previous_totals = total_token_usage`（并写回折叠快照）。
- **rate_limits 独立采集（二轮审查 §2）**：配额快照与用量事件是**两条独立消费链**——只要 `token_count` 事件携带 `rate_limits` 就落 `quota_snapshots`，不因差分算法第 1/2/4 条跳过用量而连带跳过。limit key 稳定化：优先源原生 `limit_id`；缺省按槽位规范化为 `codex:primary` / `codex:secondary`，同一逻辑限额跨事件的键保持稳定（ADR-0008）。
- **截断/重写**：`offset > 文件大小` 或首行 meta 的 `id` 变化 → 该流 `generation++` 从头摄取，靠 `eventKey` 去重兜底。
- **文件发现**：周期扫描 `sessions/**` 与 `archived_sessions/**` 的 `*.jsonl`；归档移动表现为"旧路径消失、新路径出现"，`eventKey`/`originStream` 用**相对 sessions 根的路径**而非绝对路径，移动后相对路径不变则不产生重复。

## 去重与已知坑

1. **fork/分支会话内嵌父历史（最大坑）**：fork 出的新文件会把父会话历史完整复制进开头（含父的 `session_meta` 与全部 `token_count`），本机实测 fork 文件含 2 个不同 id 的 meta 行。不去重则父会话用量双倍计数——与 ccusage issue #19（branched conversations fully counted）同类。对策（按可靠性排序）：
   - A. 读父文件（`forked_from_id` / `thread_spawn.parent_thread_id` 定位）的用量事件序列作前缀，子文件逐条匹配跳过；父文件只取 fork 时间之前的部分；
   - B. 父文件缺失时退化为 marker 扫描：`task_started` 结束重放前缀、`inter_agent_communication_metadata`（0.143.0-alpha.15 前名 `inter_agent_communication`）且 `trigger_turn==true` 标志子代理自身 turn 开始；之前的事件全部视为继承历史；
   - C. 内容哈希 `eventKey` 全局去重兜底（跨流同键按收敛契约处理）。
2. **子代理（MultiAgent V2）**：每个子代理是独立 rollout 文件（`source.subagent.thread_spawn` 标识，depth/agent_path 可建树）。子代理自身 turn 的 token 是真实独立消耗，必须计数；只有继承前缀要剔除。参考 ccusage issue #313（sub-task token tracking）。
3. **auto-compact 合成 token_count**：自动压缩（上游触发点为 `ContextWindowExceeded`，上下文将满时自动发起）时 `set_total_tokens_full()` 会把 `total_tokens` 伪造成 context_window（`fill_to_context_window`），last 里只有 `total_tokens` 非零、**五项**计费字段（input/cached/cache_write/output/reasoning）全 0——按差分算法第 4 条自然过滤，**不要**把 total 当用量。
4. **resume 不产生重复行**：app-server 恢复会话时 token 用量只向连接重放通知、不重写 rollout（上游 `token_usage_replay.rs` 注释明确）；但 `cli_version` 会重录建会话时的原值，**禁止**用 cli_version 推断行为分支。
5. **历史边界（2025-09 上旬）**：2025-09-09 前（PR #3380 之前）rollout 只持久化 response_item/session_meta，**完全没有 token 数据**；09-09 至 09-11（PR #3444 加入 turn_context 之前）有 token_count 但无模型信息。这两段分别按"无数据"和"模型未知（model=null，pricing 落 unmatched）"处理，本机最老文件为 0.133 已全量带模型，未经本机验证更早版本。
6. **重复快照**：同一累计值可能被重复落盘（费率刷新等），靠算法第 2 条过滤；`thread_settings_applied`（0.144.0+）的 `service_tier`（priority/fast/default）可用于区分计费率，与 token 无关但影响计价。
7. **rollout 分页化（0.146.0 起，已合并，adapter 必须双格式兼容）**：上游已把 rollout 落盘从单文件纯追加演进为分页化（openai/codex [#30188](https://github.com/openai/codex/issues/30188) / [#32332](https://github.com/openai/codex/issues/32332) / [#33930](https://github.com/openai/codex/issues/33930)，2026-07 合并，随 0.146.0 发布；二轮审查 §3 确认）。这不是未来风险而是现役事实：adapter 必须同时兼容 **legacy（单文件纯追加）与 paginated 两种布局**，按文件布局判别格式代际并记入 `source_instances.format_version`；paginated 布局下 `streamKey` 指向页序列、`sourcePosition` 按页内偏移定义。两种布局各出 golden fixtures（M0.5 冻结门交付物），页格式细节以 fixtures 冻结为准。

## 能力声明

- 可提供：每次 API 调用的 input/cacheRead/cacheWrite/output/reasoning token、会话/父子会话树、项目路径、逐轮模型、精确到毫秒的时间戳、CLI 版本、官方 rate_limits 配额快照。
- 历史深度：2025-09-09 起有 token 数据；2025-09-11 起有模型归属；更早文件只能列出会话存在性。
- 成本：**computed**（无原生成本；按模型 + `service_tier` 查价格表；模型缺失行落 `unmatched`）。
- 本机验证覆盖 cli 0.133–0.146（legacy 布局）；paginated 布局与 `cache_write_input_tokens`、多代理字段等变体依据上游源码与 ccusage 实现，paginated 的 golden fixtures 列入 M0.5 冻结门。

## 参考链接

- 上游协议定义（TokenUsage / TokenCountEvent / SessionMeta）：<https://github.com/openai/codex/blob/main/codex-rs/protocol/src/protocol.rs>
- 落盘白名单（policy.rs）：<https://github.com/openai/codex/blob/main/codex-rs/rollout/src/policy.rs>
- token_count 持久化引入 PR #3380（2025-09-09）/ turn_context PR #3444（2025-09-11）：<https://github.com/openai/codex/commits/main/codex-rs/core/src/rollout/policy.rs>
- resume 重放不重写 rollout：<https://github.com/openai/codex/blob/main/codex-rs/app-server/src/request_processors/token_usage_replay.rs>
- 归档实现：<https://github.com/openai/codex/blob/main/codex-rs/thread-store/src/local/archive_thread.rs>
- ccusage Codex 适配器（差分/重放前缀/去重算法）：<https://github.com/ccusage/ccusage/tree/main/rust/adapters/codex> 及其文档 <https://github.com/ccusage/ccusage/blob/main/docs/guide/codex/index.md>
- 竞品教训：ccusage issue #19（分支会话重复计数）、#313（子任务 token 追踪）：<https://github.com/ccusage/ccusage/issues/19>
