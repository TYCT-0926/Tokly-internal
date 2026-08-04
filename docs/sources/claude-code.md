# Claude Code

本地 JSONL 事件流，逐 API 调用记录四分类 token（输入/输出/缓存读/缓存写）+ 模型 + 会话归属，无原生成本。数据质量评级：**丰富**（token 粒度最完整的源，唯缺原生成本与 reasoning 细分）。

## 数据位置

| 平台 | 路径 |
|---|---|
| macOS / Linux | `~/.claude/projects/<编码后的项目路径>/<sessionUuid>.jsonl` |
| Windows | `%USERPROFILE%\.claude\projects\<编码后的项目路径>\<sessionUuid>.jsonl` |
| 自定义根目录 | 环境变量 `CLAUDE_CONFIG_DIR` 存在时替换 `~/.claude`（只读探测，不得设置） |

- 项目目录名编码规则：项目绝对路径中**所有非字母数字字符替换为 `-**`。例：Windows `C:\Users\foo\my.proj` → `C--Users-foo-my-proj`（冒号、反斜杠、点全部变成 `-`）。
- 编码不可逆且**有歧义**（`.`、`_`、空格、`/` 都映射为 `-`），adapter **禁止反解目录名**，项目归属一律取事件内的 `cwd` 字段，目录名仅作兜底。
- 子代理目录：`<项目目录>/<sessionUuid>/subagents/agent-<agentId>.jsonl` + 同名 `.meta.json`（见下文）。
- 相邻但与本源无关的文件：`~/.claude/history.jsonl`（prompt 历史，字段 `display/timestamp/project/sessionId`，无 token，可选用于提示词展示）、`todos/`、`shell-snapshots/`、`file-history/` 等，adapter 忽略。

## 格式详解

每个 `.jsonl` 文件是一个会话的**纯追加**事件流，每行一个 JSON 对象。文件名 uuid = 事件内 `sessionId`（本机已验证一致）。

### 事件类型清单

**白名单原则：只消费 `type == "assistant"` 且 `message.usage` 为对象的事件。** 版本间类型增减频繁，黑名单必然漏掉新类型。

**跳过与留证的分界（2026-08-04 阶段三B 复审裁决，消解本行与「已验证格式版本集合」节的冲突）**：
- `type != "assistant"`、或 assistant 行 **`message.usage` 完全缺失** → **跳过**，不留证（不是格式演进信号，只是无 token 的行）；
- assistant 行 **`usage` 存在但不是对象**、或 usage 内已知 token 字段非整数 → **不跳过**：按可解析部分入库、`status='excluded'`、`excludeReason='unverified-schema'` + 记 `ingest_errors`（见「已验证格式版本集合」节与 ADR-0014）。

本行原文的"其余全部跳过"写在 ADR-0014 之前，与后补的 unverified-schema 机制冲突。裁 unverified-schema 胜：源 30 天即删，**静默跳过等于永久不可见的丢失**，而"存在但形状变了"正是格式演进的典型信号——少算必须可见（ADR-0014「少算是可见的，错算是不可见的」）。

| type | 处理 | 说明 |
|---|---|---|
| `assistant` | **消费** | 唯一携带 `message.usage` 的类型；`isApiErrorMessage == true` 或 `message.model == "<synthetic>"` 的个体跳过 |
| `user` | 可选消费 | 真实 prompt 或 `tool_result`（带 `toolUseResult` 字段），无 token；用于工具调用归因 |
| `summary` | 跳过 | 旧版本首行的会话摘要（`summary`/`leafUuid`）；v2.1.x 本机样本中已不再出现 |
| `system` | 跳过 | `subtype` 有 `compact_boundary`/`turn_duration`/`stop_hook_summary`/`local_command` 等，无 token |
| `attachment`、`queue-operation` | 跳过 | 附件注入、消息排队操作 |
| `file-history-snapshot`、`file-history-delta` | 跳过 | 文件检查点（/rewind 用），体积大，解析器应快速略过 |
| `mode`、`permission-mode`、`ai-title`、`last-prompt`、`pr-link`、`relocated`、`worktree-state`、`frame-link` | 跳过 | v2.1.x 本机样本观察到的其他类型，均无 token |

### token 字段的确切位置（assistant 事件）

```jsonc
// 以下为虚构示例，仅示意结构
{
  "type": "assistant",
  "uuid": "9f3e2a10-1111-4abc-8def-000000000001",   // 行级 uuid：每行唯一，但同一逻辑响应可重复出现（见去重节）
  "parentUuid": "9f3e2a10-1111-4abc-8def-000000000000",
  "sessionId": "aaaaaaaa-bbbb-4ccc-8ddd-eeeeeeeeeeee",
  "requestId": "req_011EXAMPLEabcdefghijkl",          // 顶层 API 请求 ID
  "timestamp": "2026-07-20T08:15:30.123Z",            // ISO 8601
  "cwd": "E:/work/example-shop",                      // 项目归属以此为准
  "gitBranch": "feat/cart",
  "version": "2.1.208",
  "isSidechain": false,
  "message": {
    "id": "msg_011EXAMPLEmnopqrstuv",                 // API 消息 ID（去重键的一半）
    "role": "assistant",
    "model": "claude-opus-4-8",                       // 模型字段；"<synthetic>" 需过滤
    "content": [
      { "type": "text", "text": "（虚构示例文本）" },
      { "type": "thinking", "thinking": "（虚构）" },   // thinking 无独立 token 计数，计入 output_tokens
      { "type": "tool_use", "id": "toolu_01EXAMPLE", "name": "Read", "input": {} }
    ],
    "stop_reason": "end_turn",
    "usage": {
      "input_tokens": 523,                 // → tokens.input（不含任何缓存部分）
      "output_tokens": 87,                 // → tokens.output（含 thinking）
      "cache_read_input_tokens": 28741,    // → tokens.cacheRead
      "cache_creation_input_tokens": 1204, // → cacheWrite 总量（aggregate，拆档裁决见映射表）
      "cache_creation": { "ephemeral_5m_input_tokens": 1204, "ephemeral_1h_input_tokens": 0 }, // 5m/1h 明细；与 aggregate 的一致性裁决见映射表，1h 定价不同
      "server_tool_use": { "web_search_requests": 0, "web_fetch_requests": 0 }, // 服务端工具计费次数（可选）→ event_charges 子表
      "service_tier": "standard",
      "output_tokens_details": { "thinking_tokens": 40 }, // thinking 的独立细分（信息性；output_tokens 已含之，不另计价）。新版字段，可缺省
      "inference_geo": "us"                               // 推理地域（可选，新版字段）
    }
  }
}
```

口径要点（与 Anthropic API / ccusage 一致）：`input_tokens`、`cache_read_input_tokens`、`cache_creation_input_tokens` **三者互斥**，总输入 = 三者之和；不要把 cache 部分再加进 input。`output_tokens` 恒含 thinking（`output_tokens_details.thinking_tokens` 只是信息性细分）；`service_tier`（standard/priority 等）影响计价费率档，映射到 `UsageEvent.serviceTier`；`inference_geo` 见映射表（不映射，M1 顺带评估存档——2026-08-04 裁决收敛旧「暂只存档」措辞）。

**无原生成本字段**：整条事件链没有任何美元金额，成本必须由 Rust `pricing` crate 计算。

## 归一化映射

| UsageEvent 字段 | 取值规则 |
|---|---|
| `source` | 常量 `'claude-code'` |
| `sourceInstanceId` | 由 core 按 `(source, account, machine, dataRoot)` 解析/创建 `source_instances` 行；`dataRoot` = `~/.claude` 或 `CLAUDE_CONFIG_DIR` 的规范化路径 |
| `eventKey`（去重键） | `sha1(message.id + ":" + requestId)`；缺任一字段时回退行级 `uuid` 并标 `identityQuality='weak'`；`uuid` 也缺失则无法稳定标识，记 `ingest_errors` 不入库（二轮审查 §2）。**禁止无标注地用 uuid**（重复风险见去重节） |
| `identityQuality` | 键由 `message.id+requestId` 构成：`'strong'`；回退 uuid：`'weak'` |
| `revision` | `originStream` = 相对 projects 根的文件路径；`streamGeneration` = 文件身份代际（身份断裂 ++）；`sourcePosition` = 字节偏移。**只在同 stream 内可比**（ADR-0002 收敛契约） |
| `payloadHash` | 该行原始字节的 sha256；**子代理文件**额外把归一化后的 agent 值（来自 `.meta.json`，见「agent 归一化规则」节）以 0x1f 分隔拼入哈希输入（2026-08-03）——meta 是行外上下文，不掺入则「同 hash ⇒ 同归一化事件」不变量被打破，代际重读时来自 meta 的修正值会被收敛契约规则 4 当作重复观测永久丢弃 |
| `sessionKey` | 主会话事件：事件的 `sessionId`；子代理事件：`"<sessionId>:<agentId>"`（`agentId` 为子代理文件每行的 `agentId` 字段） |
| `parentSessionKey` | 子代理事件：事件的 `sessionId`（即主会话 uuid）；主会话事件：缺省 |
| `projectKey` / `projectLabel` | label = 事件的 `cwd`（原样保存，不做加工）；key 由 core 统一归一化 |
| `timestamp` | 事件的 `timestamp` 解析为 epoch_ms |
| `model` | `message.model` |
| `agent` | 主会话事件恒 `''`；子代理事件取同名 `.meta.json` 的 `agentType`；旧版内嵌 sidechain 事件取 `'sidechain'`。细则与降级见「agent 归一化规则」节（ADR-0018） |
| `tokens.input` | `usage.input_tokens ?? 0` |
| `tokens.output` | `usage.output_tokens ?? 0`（含 thinking，符合 ADR-0002 口径不变量） |
| `tokens.cacheRead` | `usage.cache_read_input_tokens ?? 0` |
| `tokens.cacheWrite5m` / `tokens.cacheWrite1h` | **明细与 aggregate 一致性裁决（二轮审查 §1 矩阵）**：`cache_creation` 明细存在且 `ephemeral_5m + ephemeral_1h == cache_creation_input_tokens` → 精确拆档；明细存在但**不相等** → 记 `ingest_errors`（`cache-split-mismatch`）且事件 `status='excluded'` + `excludeReason='cache-split-mismatch'`（fail-visible，禁止硬猜）；明细缺失 → 全部计入 `cacheWrite5m`（ADR-0002 默认档） |
| `tokens.reasoning` | 可选：`usage.output_tokens_details.thinking_tokens`（信息性子集，不重复计价）；缺省为 0 |
| `tokenQuality` | `'native'` |
| `serviceTier` | `usage.service_tier`，缺省 null |
| `gitBranch` | 事件的 `gitBranch`，缺省 null |
| `cliVersion` | 事件的 `version` |
| 成本列 | `costNativeMicroUSD = null`（无原生成本）；`pricingEligibility='standard'`；`displayPolicy='computed-only'`（ADR-0002 成本所有权） |
| `event_charges` | 可选：`usage.server_tool_use` 的 `web_search_requests`/`web_fetch_requests` 非零时落 `event_charges`（kind=`web-search`/`web-fetch`） |
| `inference_geo` | **不映射**（2026-08-04 裁决入表；M1 顺带评估是否存档，adapter 现状忽略） |
| `toolCall` | 可选：从 assistant `content[]` 中 `tool_use` 块取 `name`。**落 `tool_calls` 子表**（事件身份关联），不再发零 token 独立事件（ADR-0002） |

## 增量摄取策略

水位线按 ADR-0002（JSONL 类）：`watermarks` 行 = `(sourceInstanceId, 'default', 相对路径, 'jsonl-offset')` + 字节偏移 + mtime + 文件大小。

1. JSONL 纯追加 → **按字节偏移续读**。比对 mtime/大小，未变则跳过整个文件。
2. **行尾残缺处理**：追加写可能留下半行。每轮只消费到最后一个完整 `\n` 为止，JSON parse 失败的尾部字节不推进水位线，留给下一轮；**完整但非法的行**记 `ingest_errors`（错误码/hash/长度/位置）后推进水位线（ADR-0002）。
3. **截断/替换检测**：文件大小 < 水位线偏移，或文件被整体重写 → 该流 `generation++` 全量重读。幂等性由 `(sourceInstanceId, eventKey)` 主键保证，重读不产生重复。
4. **文件删除**：Claude Code 启动时按 `cleanupPeriodDays`（默认 30 天）删除旧 jsonl——本机实测最老文件恰好 30 天。adapter 发现文件消失时静默丢弃水位线，已入库数据不受影响（这正是 Tokly 本地仓库的价值；**缺席 ≠ 删除**，不标 tombstone）。
5. **无差分需求**：`usage` 是单次 API 调用的瞬时值，非累计计数器，无需相邻事件差分。
6. 扫描范围（二轮审查 §2 明写）：`projects/*/*.jsonl` 及 `projects/*/*/subagents/*.jsonl` 两层通配；`*.meta.json` 只读——其 `agentType` 字段用于 agent 归一化（见「agent 归一化规则」节），其余字段仅供会话标题/类型展示，不进用量管线。

## 去重与已知坑

1. **去重键必须是 `message.id + requestId`，不能是行级 uuid**——双重证据：
   - 本机实测：同一文件内同一 `message.id|requestId` 对出现 **3 次**（流式/重试写多行），而每行 uuid 均唯一。用 uuid 去重会把一次响应计 3 次——故 uuid 只作 `identityQuality='weak'` 的兜底键。
   - [ccusage issue #19](https://github.com/ryoppippi/ccusage/issues/19)：分支会话（Esc+Esc rewind / resume fork）会生成**新文件复制旧事件**，uuid/sessionId 全新但 `message.id + requestId` 不变。去重必须**跨文件全局**生效（Tokly 的 `(sourceInstanceId, eventKey)` 主键天然满足），否则长会话分支后成本成倍虚高。
   - **同键多行：同一 stream 内取最后源记录；禁止取 max；跨 stream 不比较 offset**（2026-08-02 二轮审查修正，本机实测 2,384 文件 / 141,830 usage 行 / 44,727 重复组 / 25,639 组 payload 不同）：一轮发现的「first-wins 低估 36,488,655 output tokens」仍成立，但「usage 随流式单调递增」被二轮证伪——**实测至少一处同文件 `output_tokens` 从 618 回落到 0**，取 max 会虚高。规则：同一 stream（同文件同 generation）内 `sourcePosition`（字节偏移）最大者即流式终态，覆盖先到行；跨 stream（fork 新文件复制旧事件）offset 不可比——payloadHash 相同去重，不同按 ADR-0002 收敛契约进 `conflict_observations` 显式处理。（另见 anthropics/claude-code#28197）
2. **子代理必须计入**：[ccusage issue #313](https://github.com/ryoppippi/ccusage/issues/313) 的教训——Task 子代理烧的是真实 token，漏掉会严重低估。两种布局都要覆盖：
   - **新版（v2.x，本机实测）**：独立文件 `<sessionUuid>/subagents/agent-<agentId>.jsonl`，每行带 `agentId`，assistant 事件 `isSidechain: true`，`sessionId` 仍为主会话 uuid。`.meta.json` 的 `{agentType, description, toolUseId, spawnDepth}` 可用于把子代理挂到主会话的 Task tool_use 上（`toolUseId` 即关联键）。
   - **旧版**：sidechain 事件内嵌主文件，`isSidechain: true`，靠 `parentUuid` 链区分支线。映射为 `sessionKey = "<sessionId>:sidechain"`。
3. **`<synthetic>` 模型与 API 错误**：`message.model == "<synthetic>"` 的 assistant 事件是本地合成的提示消息（本机实测存在），`isApiErrorMessage == true` 的是失败调用（本机实测 18 例），均不计费，过滤。
4. **compact 边界**：`system`/`compact_boundary` 事件标记上下文压缩点，无 token；会话分段展示可用，用量管线忽略。
5. **版本差异**：事件含 `version` 字段（本机样本 2.1.200–2.1.208）。旧版有 `summary` 首行、内嵌 sidechain；新版有 subagents 目录、更多辅助类型。坚持白名单 + 可选字段兜底（`?? 0`）即可兼容，不为特定版本写分支逻辑。
6. **历史深度硬上限**：`cleanupPeriodDays` 默认 30 天启动即删，不可恢复（[claude-code#62476](https://github.com/anthropics/claude-code/issues/62476)、[官方 settings 文档](https://docs.anthropic.com/en/docs/claude-code/settings)）。首次导入必须全量回填本地已有全部历史，之后增量保鲜；UI 不得承诺超过本地文件的历史。

## 已验证格式版本集合（2026-08-03）

- **已验证集合（结构形态定义）**：`type == "assistant"` 且 `message.usage` 为对象、token 四计数字段（`input_tokens` / `output_tokens` / `cache_read_input_tokens` / `cache_creation_input_tokens`）为可选非负整数的行结构。实测覆盖 Claude Code **v2.1.200–2.1.208**（2026-08-03，本机 35 项目 / 106 jsonl 只读抽样）。
- **本源无独立格式版本字段**：事件内 `version` 是 CLI 版本而非格式版本，按映射表落 `cliVersion`（`rawSchema` 留空——2026-08-04 裁决以映射表为准收敛本节旧措辞）仅供排查，**不作为 unverified-schema 的触发条件**——否则源工具每次发版都会把新用量整批 excluded，与白名单 + 可选字段兜底的兼容策略（见「去重与已知坑」5）矛盾。
- **unverified-schema 触发条件（结构性，ADR-0014）**：`type == "assistant"` 且 `message.usage` 存在但**不是对象**，或 usage 内已知 token 字段出现非整数类型——事件按可解析部分入库、`status='excluded'`、`excludeReason='unverified-schema'`，记 `ingest_errors`（错误码 `unverified-schema`，`detail_key` 记 cliVersion）。usage 完全缺失的 assistant 行按既有白名单规则跳过，不属格式演进信号。
- 集合演进：每支持一个新结构形态，本节与 golden fixtures 同批更新（ADR-0014）。

## agent 归一化规则（ADR-0018，2026-08-03）

| 事件来源 | `agent` 取值 |
|---|---|
| 主会话文件（`projects/*/*.jsonl`）非 sidechain 事件 | `''`（无智能体） |
| 子代理文件（`*/subagents/agent-<agentId>.jsonl`） | 同名 `.meta.json` 的 `agentType`（如 `code-reviewer`）——低基数、可聚合；**不用 `agentId`**（每次派生唯一，会把永久聚合键炸成高基数，违反 ADR-0018 预算前提） |
| 子代理文件但 `.meta.json` 缺失或无 `agentType` | `'subagent'`（显式未知型占位——不用 `''`：空串会把子代理用量静默并进主会话桶，丢掉 ccusage #313 教训里的维度） |
| 旧版内嵌 sidechain 事件（`isSidechain: true` 于主文件内） | `'sidechain'` |

## 能力声明

| 能力 | 支持 | 备注 |
|---|---|---|
| 逐消息 token（input/output/cacheRead/cacheWrite） | ✅ | 四分类齐全，API 原生口径 |
| reasoning token | ❌ | thinking 计入 output，无独立计数 |
| 模型 | ✅ | `message.model` |
| 时间戳 / 项目 / git 分支 | ✅ | `timestamp` / `cwd` / `gitBranch` |
| 工具调用 | ✅ | assistant `tool_use` 块 + user `toolUseResult` + 子代理 meta |
| 子代理用量 | ✅ | subagents 目录（新版）/ isSidechain（旧版） |
| 智能体维度（agent） | ✅ | `agentType`（meta）归一化，规则见「agent 归一化规则」节 |
| 成本 | computed | 无原生字段；`usage.cache_creation` 的 5m/1h 细分（一致性裁决后）+ `server_tool_use` 次数可支持精细计价 |
| 历史深度 | 受源限制 | 默认仅最近 30 天（`cleanupPeriodDays`），用户调大则更深 |

## 参考链接

- [ccusage issue #19 — 分支会话重复计数](https://github.com/ryoppippi/ccusage/issues/19)
- [ccusage issue #313 — 子代理 token 漏计](https://github.com/ryoppippi/ccusage/issues/313)
- [ccusage 仓库（去重与解析参考实现）](https://github.com/ryoppippi/ccusage)
- [Claude Code 官方 settings 文档（cleanupPeriodDays 等）](https://docs.anthropic.com/en/docs/claude-code/settings)
- [claude-code issue #62476 — 30 天静默删除 transcript](https://github.com/anthropics/claude-code/issues/62476)
- [LiteLLM 价格表（computed 成本来源）](https://github.com/BerriAI/litellm/blob/main/model_prices_and_context_window.json)

> 本文档格式细节经本机只读抽样验证（Claude Code v2.1.200–2.1.208，Windows，35 个项目目录 / 106 个 jsonl）；示例均为虚构脱敏数据。

