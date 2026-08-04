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
