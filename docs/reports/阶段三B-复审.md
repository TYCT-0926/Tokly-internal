不通过（3 组必须修复）

# 阶段三B · 冻结收口复审

- 日期：2026-08-04
- 对象：公开分支 `feat/freeze-closure`，复审入口 `cfa53da`，基线 `origin/main@02603ec`；冻结 fixtures 提交 `1e2652c`
- 范围：六源 golden fixtures、数据门裁决收尾、DDL 三方一致性、Claude 真实 adapter 接线、公开产出卫生
- 边界：未合并、未碰 main；三项阻塞均属冻结契约或真实 adapter 行为，不代改。仅就地清理 6 条已被 source spec 定稿的陈旧 `unresolved` 声明，并同步相邻说明，见「小修记录」

## 一、结论与必须修复

全部验收命令通过，DDL 文档门在持有 `internal/` 的工作树中真实执行且 0 skipped；四源九个 case 的 token 算术、时间戳、成本、归属、状态、覆盖与子集合抽查也都对拍。结论仍为不通过，因为绿色门禁未覆盖以下三组冻结纪律缺陷。

### 必须修复 1：Cursor `eventKey` 对照 spec 无法唯一推导

**证据链**：

1. `internal/docs/sources/cursor.md:71` 同时允许两种值：键第三段的裸 `bubbleId`，或完整键 `bubbleId:<composerId>:<bubbleId>`；「稳妥写法」表达偏好，不构成单值契约。
2. 四个 Cursor expected 均选择完整键，且 `unresolved: []`：
   - `crates/tokly-core/tests/spec_contracts/cursor/bubble-estimated-output/expected.json:11,15`
   - `crates/tokly-core/tests/spec_contracts/cursor/bubble-native-legacy/expected.json:9,13`
   - `crates/tokly-core/tests/spec_contracts/cursor/bubble-user-input-pool/expected.json:8,12`
   - `crates/tokly-core/tests/spec_contracts/cursor/subagent-auto-mode/expected.json:10,14`
3. 按裸 ID 实现与按完整键实现都符合当前 source spec，却只会有一个通过 frozen expected；因此完整键是 fixture 自行作出的设计选择，不是从权威 spec 唯一推导的值。

**影响**：违反本轮明确判据「对照 spec 无法唯一推导却入了 golden」。先把 `cursor.md` 收敛为单值，再保留或调整 fixtures；在此之前不能签核数据门。

**交叉模型分歧**：外部 Claude 复核认为 fixture 可以自行选择完整键；本复审不采纳该意见，因为本轮裁决标准明确禁止 golden 代替 source spec 作设计决定。完整键更稳妥可以作为后续裁决理由，但不能反向证明当前 spec 已唯一。

### 必须修复 2：`UNRESOLVED` 的「不冻结」保证未被 harness 实现

**证据链**：

1. `crates/tokly-core/tests/spec_contracts/README.md:24-26` 声明：无法唯一推导的值应携带 `UNRESOLVED:` marker，且不被契约冻结。
2. `crates/tokly-core/tests/spec_contracts.rs:125-138` 先对整个事件做精确 serde 往返比较，随后只检查「出现 marker 时，`unresolved` 数组非空」。它不解析声明中的字段路径、不要求声明与 marker 一一对应，也不对声明字段做 mask。
3. 字符串以外或缺省字段无法携带 `UNRESOLVED:` marker，却仍以精确零值、精确字符串或精确缺省形态写入 expected：
   - Copilot OTEL：`tokens`、`rawSchema`、`rawUsage` 三项；
   - Copilot shutdown：`tokens`、`rawSchema` 两项；
   - Gemini fail-visible：`tokens`、`rawUsage` 两项；
   - OpenCode fail-visible：`tokens`、`rawUsage` 两项。
4. `crates/tokly-core/tests/spec_contracts/copilot/session-state-shutdown-unverified/expected.json:43` 还精确冻结了 `rawUsage` 序列化，却没有把它列进 `unresolved`。spec 只要求留存内容，没有规定键序与序列化字节，这是未标记的猜测值。
5. 定向 `spec_contracts` 2/2 与全量 nextest 65/65 均通过，说明当前门禁不能检测上述违例，而非说明这些字段已正确解冻。

**影响**：README 的「未决字段不冻结」与可执行门禁不一致；未来 adapter 可能被迫复刻占位零值、缺省形态或 JSON 键序。需选择一种可执行方案：要么 source spec 明确收敛这些值并删除 `unresolved`，要么让 harness 解析路径并在比较时 mask，同时强制每条声明与实际字段一一对应。修复时还须把 shutdown `rawUsage` 补入未决或先定 canonical serialization。

### 必须修复 3：Claude 可解析的 `output_tokens=3` 被整组清零

**证据链**：

1. `internal/docs/adr/0014-诊断与格式演进.md:18` 与 `internal/docs/sources/claude-code.md:139` 均要求 unverified schema 事件「按可解析部分入库」。
2. 合成 fixture `crates/tokly-adapter-claude/tests/fixtures/root/projects/-home-dev-proj/44444444-4444-4444-8444-444444444444.jsonl:7` 为 `input_tokens="12"`、`output_tokens=3`；输出字段是可解析的非负整数。
3. `crates/tokly-adapter-claude/src/lib.rs:409-447` 用一个四元组 `Result` 解析四个 token 字段，任一字段失败便构造 `UsageTokens::zero()`，把有效的 output 一并丢弃。
4. `crates/tokly-adapter-claude/tests/scan.rs:381-412` 明确断言两维都为 0；`crates/tokly-adapter-claude/tests/snapshots/scan__full_scan_maps_the_fixture_surface.snap:430` 同样冻结 `tokens_output=0`。
5. Claude adapter 12/12 通过，证明测试正锁定该错误行为。

**影响**：实现与 ADR/source spec 的可解析部分契约相反。应按字段保留可解析非负整数，本例事件仍为 excluded，但 `tokens.output` 应为 3；同步更新测试与快照。该修复会改已冻结期望值，依本轮纪律仅反馈，不在复审中代改。

## 二、冻结数值独立对拍

复审不信构造笔记，直接从 `internal/docs/sources/` 的映射、方程和失败策略手工推导，再与 expected 比较。实际抽查为四源九案，超过任务要求的三源各两案，并完整覆盖 Gemini 三分支。

### 2.1 Codex

| case | 独立推导的完整事件形状 | 对拍 |
|---|---|---|
| `legacy-differential` call 1 | `source=codex`；`eventKey=UNRESOLVED:content-hash-call-1`；strong；session `c0dec0de-...0001`；project label/key `/home/dev/proj`；`timestamp=1784534410000`；model `gpt-5.2-codex`；agent 空；counted/event；顺序 clamp：read=`clamp(4000,0,5200)=4000`，write=`clamp(0,0,1200)=0`，故 tokens=`1200/150/4000/0/0/40`；native、standard、computed-only；两个子集合空；其余可选字段缺省 | 逐字段一致；`eventKey` 编码确实不可唯一推导并已标 marker |
| `legacy-differential` call 2 | 同一会话/项目/model；`timestamp=1784534420000`；`100/40/60` 依次 clamp：read 40、write 60、净 input 0，tokens=`0/10/40/60/0/3`；其余状态、质量、展示与 call 1 相同；下一行累计 totals 未前进，不再产生事件 | 逐字段一致，覆盖指定 `100/40/60→0` |
| `legacy-rate-limits-independent` | `info=null`、五个计费字段全 0、累计 totals 未前进三条都不产生 UsageEvent；三条各自仍产 1 个 official quota snapshot，limit key `codex-primary-2026`，水位线越过全部五行 | `events=[]` 与 side expectations 一致 |

### 2.2 OpenCode

| case | 独立推导的完整事件形状 | 对拍 |
|---|---|---|
| `gen-a-equation-unique-solution` | `I=600,O=200,R=50,CR=300,CW=100,T=1250`；四式为 800/850/1200/1250，唯一映射为 input 已净、output 不含 reasoning；归一化 tokens=`600/250/300/100/0/50`。message id 作 key；strong；session/project/model/agent/mode 均直映射；project key `d:/work/demo-app`；`timestamp=1772500100000`、duration 30000；成本 `round(.0125×1e6)=12500`，tool-computed / `opencode:models.dev`；native、standard、both；子集合空 | 逐字段一致 |
| `gen-a-basic-cost-and-tools` | cache/reasoning 全 0，四假设虽同中但映射结果唯一；tokens=`1000/200/0/0/0/0`。key/session/project/model/agent/mode 同 spec；`timestamp=1772500000000`、duration 12000；成本 `round(.04231×1e6)=42310`；tool part 展开 `{childId=call_0001,name=read,status=completed,durationMs=1500}`，title 不映射；eventCharges 空；native、standard、both | 逐字段一致 |

### 2.3 Gemini CLI

| case | 独立推导的完整事件形状 | 对拍 |
|---|---|---|
| `inclusive-cached-split` | inclusive=`12480+312+158+96=13046`，exclusive=`23286`，total 命中 inclusive，故 input 已含 cache；净 input=`12480-10240+96=2336`，output=`312+158=470`，tokens=`2336/470/10240/0/0/158`。id/session/project/model 直映射；`timestamp=1784711645311`；agent 空、channel chats、counted/event；native；coverage 仅 `cache-write`；standard/computed-only；无 rawUsage，子集合空 | 逐字段一致 |
| `disjoint-total` | inclusive=`2000+100=2100`，exclusive=`2600`，total 命中 exclusive，故 input/cache 互斥；tokens=`2000/100/500/0/0/0`。`timestamp=1784711700000`；其余身份、归属、channel、质量、coverage、定价与上一案相同 | 逐字段一致 |
| `unresolved-cache-semantics` | inclusive=`100+50+5+0=155`，exclusive=`155+30=185`，stored total=`77777`，两式均不命中，故不得猜缓存语义。id/session/project/model 直映射；`timestamp=1784711790500`；agent 空、channel chats、excluded/event，reason=`unresolved-cache-semantics`；tokens 暂全零、rawUsage 留存，两者仍属未决；native；coverage 仅 `cache-write`；standard/computed-only；子集合空；另应产生 kind=`unresolved-semantics` 的 conflict evidence | 事件形状与 side expectation 一致；零值与 rawUsage 字节仍受必须修复 2 约束 |

### 2.4 Cursor

| case | 独立推导的完整事件形状 | 对拍 |
|---|---|---|
| `bubble-user-input-pool` | tokenCount 全 0，走 estimated；type 1 文本 39 字符，只进 input：`ceil(39/4)=10`，tokens=`10/0/0/0/0/0`。`timestamp=1780315260000`；model 从 composer fallback；strong、counted/event；coverage=`cache-read/cache-write/reasoning/input-low-coverage`；disabled/estimate-only；子集合空 | 除 `eventKey` 双读法外全字段一致 |
| `bubble-estimated-output` | tokenCount 全 0，走 estimated；type 2 文本 99 字符，只进 output：`ceil(99/4)=25`，tokens=`0/25/0/0/0/0`。`timestamp=1780315203911`；model 从 composer fallback；其余状态、coverage、定价与上一案相同 | 除 `eventKey` 双读法外全字段一致 |

Cursor 两案的数值、分池、归属与覆盖均正确；失败只在 source spec 同时允许裸 ID 与完整键，而 golden 未标未决。

## 三、`UNRESOLVED` 纪律核验

### 3.1 总账

- 复审入口 `cfa53da`：18 条 `unresolved` 声明。
- 就地小修删除 6 条已经能从裁决后 source spec 唯一推出的陈旧声明：Codex `identityQuality` 3 条、Copilot `projectLabel`/`periodEnd` 2 条、OpenCode `tokenQuality` 1 条。
- 终态工作树：12 条声明；字段值级 `UNRESOLVED:` marker 共 4 个，均为 Codex `eventKey`，由 3 条声明覆盖。

| 组 | 数量 | 复核结论 |
|---|---:|---|
| Codex `eventKey` | 3 | 有 marker；内容哈希字段集已定，但 timestamp 形态、差分/累计与编码细节未定，确实不能唯一推导，保留 |
| Copilot OTEL `tokens/rawSchema/rawUsage` | 3 | 语义仍未决，但数值零、字段缺省与序列化字符串没有 marker，也未被 harness mask，实际仍冻结 |
| Copilot shutdown `tokens/rawSchema` | 2 | 同上；另有精确 `rawUsage` 未声明，构成未标记猜测 |
| Gemini fail-visible `tokens/rawUsage` | 2 | 语义仍未决，但零值与 JSON 序列化实际仍冻结 |
| OpenCode fail-visible `tokens/rawUsage` | 2 | 语义仍未决，但零值与 JSON 序列化实际仍冻结 |

### 3.2 反向验证

1. **Codex `identityQuality`**：`internal/docs/sources/codex-cli.md:78` 已唯一规定内容哈希设计键为 `strong`；原 3 条声明过度保守，非阻塞，已删。
2. **Copilot `projectLabel` / `periodEnd`**：`internal/docs/sources/copilot.md:115,120` 已唯一规定 label 来自 `context.cwd`、periodEnd 为 shutdown 时刻；原 2 条声明过度保守，非阻塞，已删，并把合成 label 落为 `/home/dev/proj`。
3. **OpenCode fail-visible `tokenQuality`**：`internal/docs/sources/opencode.md:93` 已唯一规定命中与排除行均为 `native`；原声明过度保守，非阻塞，已删。

反向验证证明「过度保守」本身可小修；阻塞来自相反方向：Cursor 未标却双读，以及 Copilot shutdown `rawUsage` 未声明却精确冻结，同时 path-aware 解冻机制缺失。

## 四、裁决落地核验

### 4.1 `WITHOUT ROWID` 三方一致

| 事实源 | 证据 | 结果 |
|---|---|---|
| migration | `crates/tokly-core/src/migration.rs:188-212` | `tool_calls` 与 `event_charges` 均为 `STRICT, WITHOUT ROWID` |
| schema 文档 | `internal/docs/数据库-schema.md:250-282` | 两表 DDL 与 migration 一致，并记录 2026-08-04 spike 理由 |
| 一致性测试 | `crates/tokly-core/tests/ddl_consistency.rs:116-160` | 文档 DDL 建库后与 migration 的全部 `sqlite_schema` 对象规范化比较；PRAGMA 文档门独立执行 |

真实执行证据：`cargo nextest run --locked -p tokly-core --test ddl_consistency --success-output immediate` → **5/5 passed，0 skipped**；输出明确包含 `migration_matches_canonical_schema_doc` 与 `connection_pragmas_match_canonical_doc` 的测试体，且没有 `skipped:` 通知。运行前确认规范文件为 `E:\Axiom\Tokly-freeze\internal\docs\数据库-schema.md`，长度 33,633 bytes。

### 4.2 十一项 spec 收敛与裁决表逐条对应

| 裁决项 | 落地点 | 结果 |
|---|---|---|
| Codex eventKey 单值为内容哈希 | `internal/docs/sources/codex-cli.md:77` | 已落；字节编码继续明确未决 |
| Codex identityQuality strong | `internal/docs/sources/codex-cli.md:78` | 已落 |
| Codex cliVersion 不映射 | `internal/docs/sources/codex-cli.md:94` | 已落 |
| OpenCode 唯一性按映射结果 | `internal/docs/sources/opencode.md:104` | 已落 |
| OpenCode tool title 不映射 | `internal/docs/sources/opencode.md:97` | 已落 |
| 三源排除行 tokenQuality=native | `internal/docs/sources/opencode.md:93`、`copilot.md:142`、`gemini-cli.md:87` | 已落 |
| Copilot label=context.cwd | `internal/docs/sources/copilot.md:120` | 已落 |
| Copilot periodEnd=shutdown | `internal/docs/sources/copilot.md:115` | 已落 |
| Gemini 示例 total=13046 | `internal/docs/sources/gemini-cli.md:52` | 已落 |
| Gemini counted 行 rawUsage 留存裁决 | `internal/docs/sources/gemini-cli.md:62` | 已落 |
| Claude version/inference_geo 措辞收敛 | `internal/docs/sources/claude-code.md:107,138` | 已落 |

### 4.3 已冻结 fixtures 差异

- `git diff 1e2652c..cfa53da -- crates/tokly-core/tests/spec_contracts`：**空**。裁决收尾提交只改 `domain.rs` 与 `migration.rs`，未动任何 golden 数值或说明。
- 本次复审小修另计：8 文件共 8 insertions / 24 deletions，删除 6 条陈旧未决声明并同步 README；唯一事件字段变化为 Copilot 合成 `projectLabel` 从占位 marker 改成 spec 已定的 `/home/dev/proj`。token、成本、时间戳、periodEnd、eventKey、identityQuality 等冻结值零改动；独立新上下文终审通过。

## 五、Claude 真实 adapter 接线

- `cargo nextest run --locked -p tokly-adapter-claude --test scan`：**12/12 passed，0 skipped**。
- 弱键抽查：`scan__full_scan_maps_the_fixture_surface.snap:434` 为 requestId 缺失时 uuid fallback，`identity_quality=weak`，reasoning=40，service tier priority；与 `claude-code.md:86-87` 一致。
- subagent meta 缺失抽查：快照 `:436` 把 agent 降级为 `subagent`，session 后缀与父会话归属完整；与 source spec 的非空降级规则一致。
- 全字段面声明除「可解析 output 被整组清零」这一项外无新增不一致；该项已列为必须修复 3。

## 六、验收命令

| 命令 | 结果 |
|---|---|
| `cargo fmt --all -- --check` | exit 0 |
| `cargo clippy --locked --workspace --all-targets --all-features -- -D warnings` | exit 0 |
| `cargo nextest run --locked -p tokly-core --test spec_contracts` | 2/2 passed，0 skipped |
| `cargo nextest run --locked -p tokly-adapter-claude --test scan` | 12/12 passed，0 skipped |
| `cargo nextest run --locked -p tokly-core --test ddl_consistency --success-output immediate` | 5/5 passed，0 skipped；文档两项真实执行 |
| `cargo nextest run --locked --workspace --all-features` | 65/65 passed，0 skipped |
| `cargo deny check` | exit 0；advisories/bans/licenses/sources 四项 ok。输出含既有 duplicate/unmatched-allowance warning，不构成 gate failure，本批未改依赖 |
| `cargo run -p xtask -- ci` | exit 0；内部再次 fmt + clippy + nextest 65/65 + workspace all-target build |

## 七、卫生与红线

- `git config user.email` 为 `TYCT-0926@users.noreply.github.com`；`origin/main..HEAD` 非 noreply commit 数为 0。
- commit 正文署名/生成标记扫描 0 命中；本次提交不含工具署名。
- `git ls-files internal docs AGENTS.md CLAUDE.md` 为 0；`git diff --name-only origin/main` 的私有路径命中为 0。
- fixtures 目录 CJK 字面量为 0；真实用户路径/用户名候选（`C:\Users\`、`/Users/`、`E:\Axiom`、本机用户名）为 0。`/home/dev/...` 与 `D:\work\...` 均为人工合成路径。
- `git diff --check` exit 0。
- 全程只读用户工具数据；未修改任何外部工具配置、代理或环境变量；未合并、未碰 main 与他人分支。

## 八、小修记录与过门条件

### 已就地修复的小问题

1. Codex 三案删除已被 `codex-cli.md:78` 定稿的 `identityQuality` 未决声明；README 同步。
2. Copilot shutdown 删除已被 `copilot.md:115,120` 定稿的 `periodEnd`/`projectLabel` 未决声明，并把合成 label 改为 `/home/dev/proj`；README 同步。
3. OpenCode fail-visible 删除已被 `opencode.md:93` 定稿的 `tokenQuality` 未决声明；README 唯一性说明同步裁决后 spec。
4. 小修经 `git diff --check`、spec contracts 2/2、全量 nextest 65/65、clippy 与独立新上下文 diff review 验证。

### 重新签核前必须完成

1. Cursor source spec 把 eventKey 收敛为裸 ID 或完整键之一，再让四案与单值契约一致。
2. 建立 path-aware 的 `UNRESOLVED` 一一对应与 mask，或逐项收敛 source spec 后删除未决；补上 Copilot shutdown `rawUsage` 的未决/规范化裁决。
3. Claude unverified token 改为逐字段保留可解析部分；本例 output 应保留 3，并更新 12 项测试中的对应断言与快照。
4. 修复后重跑本报告「验收命令」全套，并对改动后的 golden 做新上下文复推。

在以上三组问题关闭前，数据门不得签核。

## 全分支复审

不通过（1 组必须修复：事故恢复后的文档与代码分歧，按最高级事故纪律停止复审）

- 日期：2026-08-04
- 对象：`feat/freeze-closure@2259ae5`，四笔提交顺序 `1e2652c → cfa53da → 0d60f24 → 2259ae5`；公开分支与 `origin/feat/freeze-closure` 同步且工作树干净；本复审未切换或修改 `main`。
- 恢复说明：本文件曾随 `internal/` 事故丢失；本节追加前，先从事故前会话 JSONL 中实际执行的 `Add File` 与后续 `Update File` 补丁逐行恢复上一版复审正文（196 行），没有按摘要改写旧结论。

### 必须修复：Claude source spec 自相矛盾，且实现与白名单条款冲突

**证据链**：

1. `internal/docs/sources/claude-code.md:24` 的白名单原则是绝对表述：只消费 `type == "assistant"` 且 `message.usage` 为对象的事件，**其余全部跳过**。
2. 同一 source spec 的 `:139` 又规定：`type == "assistant"` 且 usage 存在但不是对象时，事件须按可解析部分入库为 `excluded / unverified-schema` 并记 `ingest_errors`。对 `msg_J` 的 `usage: 7`，两条分别要求“不产生事件”和“产生 excluded 事件”，不能同时满足。
3. 当前实现选择第二条：`crates/tokly-adapter-claude/src/lib.rs:383-414` 对非对象 usage 构造零 token、四维 `tokenCoverage` 的 excluded 事件并返回 `EventWithError`；`tests/scan.rs:381-408` 明确断言 `msg_J` 入库且 `rawUsage = "7"`，insta 快照也冻结该事件。
4. `internal/docs/adr/0014-诊断与格式演进.md:18` 支持“按可解析部分入库”，但不能由复审者据此静默废除 source spec `:24`；项目红线明确 adapter 以 source spec 为施工权威，且发现文档与实现矛盾须停下上报而非择一。

**裁决**：实现与 `:139`、ADR-0014、测试一致，但与 `:24` 矛盾；这不授权复审者自行选定哪侧为准。这是红线违例，属于本轮允许列为“必须修复”的范围。维护者须先把 `claude-code.md:24` 与 `:139` 收敛成唯一处置，并让实现、测试和快照服从该单值契约；本复审不代改，不以现有测试绿色替代裁决。无工具跨模型对抗复核独立确认只有此项触发停审纪律。

### 事故恢复后的陈旧规范措辞（非阻塞）

- `internal/docs/数据库-schema.md:481` 仍以未来态要求 `tool_calls`、`event_charges`、`session_model_stats`、`watermarks` 四张窄表在 M0.5 spike ① 一并实测；同文后续闭环表 `:504`、`internal/docs/reports/阶段三B-裁决.md:9` 与冻结收口报告已明确只实测并采纳前两表，后两表“未实测不动”。按后文显式闭环可以唯一消解，不构成第二条红线；`:481` 是事故恢复后未清除的陈旧待办措辞。
- SQL 本体的静态核对未发现三方漂移：schema 与 `migration.rs:169-240` 都只有 `tool_calls`、`event_charges` 为 `STRICT, WITHOUT ROWID`，`session_model_stats`、`watermarks` 仍为 `STRICT`；`ddl_consistency.rs:116-154` 会逐对象比较完整 `sqlite_schema`。但因最高级事故已先触发，本轮没有继续执行 DDL 文档门，不能把静态核对冒充“真实执行通过”。
- `internal/docs/adr/0002-数据架构.md:125` 的“observation 永久保留”虽与现行 90 天保留相反，但同文 `:133` 明确称其为旧无条件表述并收窄为仅因保留期清理；按 ADR 只追加纪律可确定后文取代前文，因此本次不另列阻塞。

### 停审边界与未完成项

- 触发事故后立即停止，未执行三个 harness 突变、全分支数值复推、UNRESOLVED 反向抽查、fmt/clippy/nextest/deny/xtask ci、DDL 两项真实执行及最终卫生签核；旧章节记录的是上一轮 `cfa53da/0d60f24` 的历史证据，不代表本轮 `2259ae5` 已验收。
- 停审前只完成了分支/提交序列、事故前报告恢复、二轮裁决入口及上述文档与实现证据的只读核对。没有修改公开代码、fixtures、source spec、schema 或裁决文档；没有小问题代修。
- 文档冲突收敛后，须从本任务步骤 0 重新开始，完整跑三次 mask 突变、三源各两案逐字段手推、UNRESOLVED 核验、DDL 两项真实执行、全套门禁与卫生检查，再给数据门三档终局结论。

## 三轮裁决后续审

通过

- 日期：2026-08-04
- 对象：`feat/freeze-closure@2259ae5`，公开分支与 `origin/feat/freeze-closure` 同步且工作树干净；本轮不切换、不修改 `main`。
- 历史边界：上一节记录的是发现契约自相矛盾后立即停审的当时事实，按档案纪律不改写。维护者随后在 `阶段三B-裁决.md`「三轮裁决」把 Claude 分界收敛为单值，本节从该停审点续跑并给出终局结论。
- 小修：本轮无新增小问题需要就地修改；公开代码、fixtures、source spec、schema 与裁决文档均未改动。

### 一、三轮裁决与二轮三项修复

1. **Claude 分界已单值化且实现服从 spec**：`type != "assistant"` 或 assistant 行完全缺失 `message.usage` 时跳过；usage 已存在但非对象、或已知 token 字段非整数时，按可解析部分入库为 `excluded / unverified-schema` 并记 `ingest_errors`。修订后的 `claude-code.md:24-30,143-145` 与 `lib.rs:320-333,371-435,481-517` 一致，不再存在上一节的文档与代码分歧。
2. **Cursor eventKey**：`cursor.md:71` 已定单值全键 `bubbleId:<composerId>:<bubbleId>`；四份 expected 均为该形态，且 composer 段与 `sessionKey` 一致。`git diff --exit-code 1e2652c 2259ae5 -- crates/tokly-core/tests/spec_contracts/cursor` 返回 0，四案零改动。
3. **harness mask**：`unresolved` 路径解析为 `events[<i|*>].<field>`；未知字段、坏索引与坏语法直接失败；比较前从 expected 与 serde 回写两侧摘除声明字段；未声明字段携带 `UNRESOLVED:` marker 直接失败。mask 只解冻声明字段，不绕过 `UsageEvent` 反序列化与领域不变量。
4. **shutdown `rawUsage`**：Copilot spec 只要求最后一条 shutdown 胜出并留存 raw usage，未规定 JSON 截取边界、键序、空白或 canonical serializer；输入内容可推导，但精确字符串不可唯一推导。因此 `events[0].rawUsage` 加入 `unresolved` 并整字段 mask，符合「哪个可从 spec 唯一推导」判据。
5. **Claude 逐字段保留**：msg_K 的 `input_tokens="12"` 不可解析、`output_tokens=3` 可解析，终态为 `tokens.output=3`、`tokenCoverage=["input"]`；usage 非对象的 msg_J 无任何可解析 token 维度，四维 coverage 全标。adapter 定向测试 **12/12 passed，0 skipped**（执行 ID `42654cf7-6976-4941-802f-c7dc0d870074`）。
6. **快照边界**：`git diff -U0 0d60f24..2259ae5` 的 Claude 快照只改 msg_J/msg_K 的事件表示与同两事件的 `usage_events` 投影行；无第三事件，`sessions` / `session_model_stats` 等派生聚合零 diff。Claude fixture 输入本体在 `1e2652c..2259ae5` 零 diff。

### 二、mask 三个突变

| 突变 | 预期 | 实际 |
|---|---|---|
| 在未声明字段写 `UNRESOLVED:` marker | 红 | 红：`eventKey carries UNRESOLVED markers without a declaration`（执行 ID `1e4e6af3…`） |
| 声明 `events[0].notAUsageEventField` | 红 | 红：`unresolved path targets unknown field "notAUsageEventField"`（执行 ID `51977882…`） |
| 把已 mask 的 `events[0].tokens.input` 从 `0` 改为 `17` | 绿 | 绿，1 passed（执行 ID `90aba928…`） |

补充反证：把已 mask 的事件改成 `reasoning=17, output=0` 仍先被领域校验拒绝，证明 mask 不能隐藏非法 `UsageEvent`。恢复基线后复跑为绿（执行 ID `06c81d33…`）；突变只发生在 detached 临时 worktree，公开分支始终未改。

### 三、冻结数值独立对拍

以下先从各案 input 与 source spec 手工推导完整 `UsageEvent`，再读 expected 对拍；不以 expected 的 notes 作为推导依据。实际覆盖四源九案，超过「三源各两案」最低要求，并覆盖指定四个重点。

#### 3.1 Codex

| case | 独立推导 | 对拍 |
|---|---|---|
| `legacy-differential` call 1 | `source=codex`；内容哈希键编码未定故 marker；strong；session `c0dec0de-…0001`；项目 `/home/dev/proj`；`timestamp=1784534410000`；model `gpt-5.2-codex`；agent 空；counted/event。顺序 clamp：read=`clamp(4000,0,5200)=4000`，write=`clamp(0,0,1200)=0`，tokens=`1200/150/4000/0/0/40`；native、standard、computed-only；可选字段缺省、两个子集合空 | 全字段一致 |
| `legacy-differential` call 2 | 同会话/项目/model；`timestamp=1784534420000`；对 `100/40/60` 先得 read=40，再在余量 60 上得 write=60，净 input=0；tokens=`0/10/40/60/0/3`；其余状态与 call 1 相同。下一行累计 totals 未前进，不产生第三事件 | 全字段一致，覆盖 `100/40/60→0` |
| `legacy-rate-limits-independent` | `info=null`、五个计费字段全零、累计 totals 未前进三种情形均不产 UsageEvent，但三行各自仍产 official quota snapshot，limit key 来自 `codex-primary-2026` | `events=[]` 与 side expectation 一致 |

#### 3.2 OpenCode

| case | 独立推导 | 对拍 |
|---|---|---|
| `gen-a-equation-unique-solution` | `I=600,O=200,R=50,CR=300,CW=100,T=1250`；四式依次为 `800/850/1200/1250`，唯一映射为 input 已净、output 不含 reasoning；故 tokens=`600/250/300/100/0/50`。message/session/project/model/agent/mode 直映射；project key `d:/work/demo-app`；`timestamp=1772500100000`、duration 30000；成本 `round(.0125×1e6)=12500`，tool-computed / `opencode:models.dev`；native、standard、both | 全字段一致 |
| `gen-a-basic-cost-and-tools` | cache/reasoning 全零时多个假设标签成立但映射结果相同，按 spec 仍是唯一解；tokens=`1000/200/0/0/0/0`。`timestamp=1772500000000`、duration 12000；成本 `round(.04231×1e6)=42310`；tool part 为 `{childId=call_0001,name=read,status=completed,durationMs=1500}`，title 不映射；其余身份、状态与成本归属直映射 | 全字段一致 |

#### 3.3 Gemini CLI

| case | 独立推导 | 对拍 |
|---|---|---|
| `inclusive-cached-split` | inclusive=`12480+312+158+96=13046`，exclusive=`23286`，total 命中 inclusive；净 input=`12480-10240+96=2336`，output=`312+158=470`，tokens=`2336/470/10240/0/0/158`。id/session/project/model 直映射；`timestamp=1784711645311`；counted/chats/event；native；coverage 仅 `cache-write`；standard/computed-only | 全字段一致 |
| `disjoint-total` | inclusive=`2000+100=2100`，exclusive=`2100+500=2600`，total 命中 exclusive；tokens=`2000/100/500/0/0/0`；`timestamp=1784711700000`；其余身份、归属、状态、coverage 与定价同上 | 全字段一致 |
| `unresolved-cache-semantics` | inclusive=`100+50+5+0=155`，exclusive=`185`，stored total=`77777`，两式均不命中；不得猜缓存语义。事件 excluded/chats/event，reason=`unresolved-cache-semantics`；token 暂零、rawUsage 留存且两字段未决；native；coverage=`cache-write`；另产 `unresolved-semantics` conflict evidence | 已冻结字段全一致；未决字段由 mask 解冻 |

#### 3.4 Cursor（额外覆盖）

| case | 独立推导 | 对拍 |
|---|---|---|
| `bubble-user-input-pool` | tokenCount 全零走 estimated；type 1 文本 39 chars，只进 input，`ceil(39/4)=10`；tokens=`10/0/0/0/0/0`；`timestamp=1780315260000`；model 从 composer fallback；coverage=`cache-read/cache-write/reasoning/input-low-coverage`；disabled/estimate-only | 全字段一致 |
| `bubble-estimated-output` | tokenCount 全零走 estimated；type 2 文本 99 chars，只进 output，`ceil(99/4)=25`；tokens=`0/25/0/0/0/0`；`timestamp=1780315203911`；其余状态、coverage 与定价同上 | 全字段一致 |

### 四、UNRESOLVED 纪律

全仓 20 份 expected 均含 `unresolved` 数组；7 案非空，共 **13 条字段声明、4 个值 marker**。marker 全为 Codex `eventKey`，由三条通配声明覆盖四个事件；其余非字符串或可缺省字段通过路径声明直接 mask。

| 组 | 声明 | 结论 |
|---|---:|---|
| Codex `eventKey` | 3 | spec 只定内容哈希字段集，timestamp 形态、差分/累计输入与字节编码仍明文待阶段八，不能唯一产生 hash |
| Copilot OTEL `tokens/rawSchema/rawUsage` | 3 | 四通道验证集为空；token 解释等版本表样本，raw schema 表示与 raw usage canonical 串均未定 |
| Copilot shutdown `tokens/rawSchema/rawUsage` | 3 | 同上；最后快照的语义内容可定，但归一化 token 零值策略及 raw 字符串表示不可定 |
| Gemini fail-visible `tokens/rawUsage` | 2 | spec 定 excluded/reason/留证，不定排除行归一化 token 值与 raw JSON 序列化 |
| OpenCode fail-visible `tokens/rawUsage` | 2 | 同上 |

反向验证三组：① Codex eventKey 编码确实不能唯一推导；② Copilot 空验证集下 tokens/rawSchema/rawUsage 均不能唯一推导；③ Gemini/OpenCode 的排除行 tokens 与 rawUsage 不能唯一推导，而两源 `tokenQuality='native'` 已由 spec 唯一定稿，相关陈旧声明在 `0d60f24` 删除且未残留。**未发现过度保守，也未发现未标记猜测值。**

字段级 JSON 比较补证：`1e2652c..0d60f24` 只有 Copilot shutdown `projectLabel` 从旧 marker 按裁决收敛为 `/home/dev/proj`，所有 token/cost/timestamp 数值零变；`0d60f24..2259ae5` 的五源 pending expected 事件字段零 diff，仅 unresolved 文案与 shutdown `rawUsage` 声明变化。

### 五、一轮裁决与 DDL

- migration 的 `tool_calls` / `event_charges` 均为 `STRICT, WITHOUT ROWID`；`session_model_stats` / `watermarks` 仍为 `STRICT`。`数据库-schema.md` SQL 块与之一致；`ddl_consistency.rs` 从真实 `internal/docs/数据库-schema.md` 取规范 DDL，逐对象比较完整 `sqlite_schema`。
- `cargo nextest run --locked -p tokly-core --test ddl_consistency --success-output immediate`：**5/5 passed，0 skipped**；`migration_matches_canonical_schema_doc` 与 `connection_pragmas_match_canonical_doc` 两个文档门均真实执行（执行 ID `732747a7-a204-4c9f-81ca-0bb40c13b263`），不是跳过态。
- schema 备注 `:492` 仍保留 spike 前的未来态四表措辞；同文 `:516` 的 2026-08-04 关闭行、裁决表和 SQL 本体明确只采纳两张孪生子表，后两表未实测不动。它是历史待办措辞，不改变 DDL 契约，不构成文档与代码分歧；建议后续文档维护时清除陈旧未来态。
- spec 收敛十一项逐条对应：Codex eventKey/identityQuality/cliVersion；OpenCode 退化唯一性/tool title；三源排除行 tokenQuality；Copilot projectLabel/periodEnd；Gemini 示例 total/折叠留存；Claude version 与 inference_geo 措辞。未发现漏项或反向实现。
- `git diff 1e2652c..cfa53da -- crates/tokly-core/tests/spec_contracts` 为空；一轮裁决落地未改 golden。Cursor 全目录 `1e2652c..2259ae5` 为空；二轮 pending expected 除裁决所需元数据/未决声明外，冻结数值零改。

### 六、验收命令

| 命令 | 结果 |
|---|---|
| `cargo fmt --all -- --check` | exit 0 |
| `cargo clippy --locked --workspace --all-targets --all-features -- -D warnings` | exit 0 |
| `cargo nextest run --locked -p tokly-core --test spec_contracts` | 2/2 passed，0 skipped（执行 ID `d322b89e-827d-46a1-b40c-32226a587185`） |
| `cargo nextest run --locked -p tokly-adapter-claude --test scan` | 12/12 passed，0 skipped（执行 ID `42654cf7-6976-4941-802f-c7dc0d870074`） |
| `cargo nextest run --locked -p tokly-core --test ddl_consistency --success-output immediate` | 5/5 passed，0 skipped；文档两项真实执行（执行 ID `732747a7-a204-4c9f-81ca-0bb40c13b263`） |
| `cargo nextest run --locked --workspace --all-features` | 65/65 passed，0 skipped（执行 ID `06ac1764-a484-47b8-82fa-033e91b73d5c`） |
| `cargo deny check` | exit 0；advisories/bans/licenses/sources 四项 ok；仅既有 allowance/重复依赖 warning |
| `cargo run -p xtask -- ci` | exit 0；内部再跑 fmt、clippy、nextest 65/65 与 all-target build；nextest 执行 ID `111b41be-9bd6-4adc-ad11-ccf1182ad7da` |

### 七、卫生与终局

- `git config user.email` 为 `TYCT-0926@users.noreply.github.com`；四笔公开提交全为该 noreply 邮箱，作者/提交者名唯一值为 `TYCT`；commit 正文无工具署名或生成标记。
- `git ls-files internal AGENTS.md CLAUDE.md` 无输出；`origin/main...HEAD` 文件名与 diff 正文均无 `internal/` 私有路径。
- fixtures 无 `E:\Axiom`、`C:\Users`、`/Users/`、本机用户名等真实路径/身份命中；`/home/dev/...` 与 `D:\work\...` 均为人工合成。fixtures 的 CJK 字面量为 0。
- `git diff --check origin/main...HEAD` exit 0；公开工作树干净，私有仓除本报告追加外无其他改动。
- 无工具外部 Claude 在三轮裁决前对同一 `2259ae5` 完整 diff 的独立复核只识别出上一节已停审的 Claude spec 冲突；该冲突现已由维护者单值裁决并由新上下文续审验证关闭。本轮两次追加外部调用分别在 304s/244s 超时且输出为空，未把超时冒充通过，也未据此新增或消除问题。

三轮裁决关闭唯一事故项后，二轮三项修复、冻结数值、UNRESOLVED 解冻纪律、DDL 三方一致性、真实 adapter 接线及全套验收均有独立证据通过；未发现新的实质问题。**终局结论：通过。**
