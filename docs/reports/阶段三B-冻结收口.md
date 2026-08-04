# 阶段三B · 冻结收口 执行报告

- 日期：2026-08-04；分支 `feat/freeze-closure`（worktree `../Tokly-freeze`，基于 main `02603ec`，提交 `1e2652c` 已推送；未碰 main）
- 范围：六源 golden fixtures（任务 A）+ 数据门评审包（任务 B：[冻结门评审-数据门-2026-08-04](../research/冻结门评审-数据门-2026-08-04.md)）
- 工具链：rustc 1.97.1（pin）/ nextest / cargo-deny，与阶段二一致

## 交付物

### A1 · claude-code fixtures（经真实 adapter，spec 全字段面）

fixtures 由 3 文件 7 事件扩至 **5 文件 13 事件**，adapter 测试 8 → **12 项**，insta 映射快照重生成并逐字段人工复核。新增覆盖（阶段二已有面不重复列）：

| 用例 | 断言点 |
|---|---|
| 弱键 fallback（无 requestId） | eventKey = 行级 uuid、`identityQuality='weak'`；同行携带 reasoning=40（≤output）与 `serviceTier='priority'` |
| unverified-schema 结构触发 ×2 | usage 非对象（`usage:7`）与 token 字段非整数（`"12"`）→ 事件入库 excluded、tokens 置零、rawUsage 留存、`ingest_errors` 双行 `detail_key='2.1.208'` |
| subagent meta 缺失 | agent 降级 `'subagent'`（永不空串），session `<sid>:d4e5f6`，进 session_model_stats 二维键 |
| 旧版内嵌 sidechain | session `<sid>:sidechain`、agent `'sidechain'`、parent 指主会话；同行 web_fetch → event_charges（web-fetch 1000 milli） |
| 跳过白名单四型 | user / isApiErrorMessage / `<synthetic>` / 无 usage → lines_skipped=4，零观测 |
| unresolved-mapping 双错误 | 坏时间戳、无稳定身份（三 id 全缺）→ 记错不入库 |
| model/cwd/version 缺失 | model=NULL（不造魔法值）、projectKey=''、cliVersion/gitBranch=NULL 照常 counted |

阶段二 TODO ①（sidechain 无 fixture）就此闭合；TODO ②–⑦ 状态不变（归 M1/spec 收敛，见评审包 §四 A.17/18）。

### A2 · 五源 spec 契约快照（`crates/tokly-core/tests/spec_contracts/`）

**20 case / 21 期望事件**，输入样本 + 手工推导期望 UsageEvent 成对入库，README 标注「待各自 adapter 里程碑接线」。harness（`spec_contracts.rs`）：域反序列化（token 不变量类型层强制）+ `validate()` + **serde 逐字节往返**（未知/拼错字段即红）+ 排除行必有 excludeReason + `UNRESOLVED:` 标记必须显式声明；经突变自证（注入拼错字段 → 红）。

| 源 | case | 冻结要点 | 未决（详见评审包 §四 A） |
|---|---|---|---|
| codex | 4 | 差分 last 优先、双减 clamp（100/40/60→净 0 上游用例）、重复快照不重计、rate_limits 三型独立采集、nickname/降级 | paginated 停下（#1）、eventKey 编码（#2）、identityQuality 无表行（#3） |
| opencode | 5 | Gen A/B、方程四假设唯一解 → output+reasoning 合并、tool-computed 成本、fail-visible、agent≠mode、rollup 非事件源 | 退化唯一性措辞（#6）、title vs ADR（#7）、排除行 tokenQuality（#8） |
| copilot | 2 | 空集 → unverified 处置（行为冻结、语义零冻结）、末快照胜出、premium-request 进 charges、[sec,nanos] 时间戳 | projectLabel 双读法打 UNRESOLVED（#10）、periodEnd（#11）、版本判别张力（#12） |
| cursor | 4 | 估算路线（分池 ceil(chars/4)、0+coverage、disabled/estimate-only）、native 遗留、子代理父链、auto→NULL | 字符抽取与 Best-of-N 词条（#14） |
| gemini-cli | 5 | inclusive/disjoint/判别失败三分支、id 折叠、model 回退、二值 agent | spec 示例 total 自相矛盾（#15）、rawUsage 留存背书（#16） |

**「期望值拿不准不猜」的执行记录**：paginated 整层停下（spec 自身未定页格式且与二轮审查记载相反）；copilot projectLabel、codex eventKey 编码等不可唯一推导字段一律 `UNRESOLVED:` 标记入库——公开仓的标记与评审包 §四 清单一一对应，harness 强制声明。

### B · 数据门评审包

[冻结门评审-数据门-2026-08-04](../research/冻结门评审-数据门-2026-08-04.md)：五件冻结物逐件状态与证据链、spike ① 验收逐条勾稽（引阶段二数字）、门序修正记录与核对、18+3 项偏差与未决项全清单、**「可过（附 3 项签核确认）」建议**与反面清单。

## 交叉验证（合并前，与构造者不同上下文）

六个独立复推 agent 并行：五源逐字段重推导（明确指令「不信构造者笔记、自己重算」）+ 公开产出卫生扫查。结果：

- **全部冻结数值零错误**——token 算术、顺序 clamp、方程两侧（唯一解与无解均独立重算）、全部 epoch-ms 时间戳、微美元舍入、估算字符数（99→25、39→10、20→5）逐项对拍成功。
- 存活发现 **1 blocker + 6 should-fix/nit**，全部属冻结纪律与 spec 措辞一致性，逐条裁决：
  1. [blocker] copilot projectLabel 冻结了 cwd 而映射行存在平行对读（→repository）——复核确认双读法真实存在且 projectKey 两读法下唯一 → **改打 `UNRESOLVED:` 标记**（不猜任何一边），评审包 §四 #10 上报 spec 定稿；
  2. codex 输入 total_tokens 算术与上游恒等式（total=input+output）不符 → **修输入**（110/5460）；
  3. codex 规则 2（重复快照）覆盖非判别性 → **补一行非零累计逐字重复**（无该守卫的 adapter 会三事件，契约钉死两事件）；
  4. codex identityQuality 无 spec 行 → 保值 + unresolved 声明 + 上报补行；
  5. cursor 输入保真度两处（unifiedMode 位置、dataRoot 布局）→ 修输入；
  6. opencode 排除行 tokenQuality / gemini rawUsage 缺省 → 补 unresolved 声明与 notes 背书；
  7. codex README paginated 措辞与 spec 记载相抵 → 公开侧改为只述「spec 延迟页格式 + 零样本」，spec↔审查矛盾移入 internal 评审包。
- 复推同时发现 **spec 自身缺陷 4 处**（opencode 退化唯一性未成文、opencode title vs ADR-0002 冲突、gemini 示例 total 不满足自身方程、claude rawSchema/cliVersion 措辞）——internal 只读红线内不代改，全部入评审包 §四 待维护者裁决。
- 卫生扫查：56 文件全读，零 internal 引用（「maintained outside the public tree」惯用语除外）、零 AI 痕迹、全虚构数据、`.jsonl/.snap` LF 钉死（.gitattributes）；一项存量观察（domain.rs:529 中文字面量，02603ec 已入）上报不动。

## 逐命令验证（修正后终态，全绿）

| 命令 | 结果 |
|---|---|
| `cargo fmt --all -- --check` | exit 0 |
| `cargo clippy --locked --workspace --all-targets --all-features -- -D warnings` | exit 0 |
| `cargo nextest run --locked --workspace --all-features` | **65/65**（阶段二关闭 59 → +2 spec_contracts harness、+4 adapter 定向）|
| `cargo build --locked --workspace --all-targets --all-features` | exit 0 |
| `cargo deny check` | advisories/bans/licenses/sources 四项 ok（零新依赖） |
| `cargo run -p xtask -- ci` | exit 0 全链 |
| DDL 一致性 | worktree 无 internal/，以 **NTFS junction 挂载后两项真实执行**（`--no-capture` 无跳过通知）；junction 用毕即移除（`rmdir` 仅删链接，internal 完好核验） |

两次突变自证：harness 注入拼错字段即红（已还原）；claude 快照与定向断言互锁。

## 工作方式与卫生

- 独立 worktree `E:\Axiom\Tokly-freeze`（feat/freeze-closure），未碰 main 与他人分支；`../Tokly-design-tokens` 未触。
- 单提交 `1e2652c`（56 文件 +2110/-15），noreply 身份，正文零 AI 痕迹；已推送 origin。
- internal/ 仅新增本报告与评审包两文件，其余只读。

## 裁决落地（2026-08-04，依 [阶段三B-裁决](阶段三B-裁决.md)，同分支收尾批）

1. **①-b 落码**：migration v1 的 `tool_calls` 与 `event_charges` 转 `STRICT, WITHOUT ROWID`（与修订后 schema 文档逐字对齐）；DDL 一致性测试对修订后文档真实通过（junction 挂载，无跳过通知）；受影响快照核验——`SELECT *` 面不受 WITHOUT ROWID 影响，adapter insta 快照与全部测试**零变化**（65/65 复跑确认）。cache_size 维持，文档已打戳，代码零改动。
2. **② spec 收敛批次十一项**已按裁决表逐条落进 sources/ 五份 spec（改动全部带 2026-08-04 裁决注记）：codex eventKey 单值化（内容哈希；`路径:偏移` 明示为 sourcePosition 非事件键；编码细则留阶段八）+ identityQuality `strong` 补行（同毫秒同参合并残余在册）+ cliVersion 显式不映射 + serviceTier 未实机注记；opencode 退化唯一性一句（唯一性定义在映射结果）+ toolCall 删 title（ADR 为准注记）+ 排除行 tokenQuality 补句；copilot label = cwd 原始值（repository 不映射）+ periodEnd = shutdown 时刻 + 排除行 tokenQuality 补句；gemini 示例 total 修 13046（自洽注记）+ counted 行不存 rawUsage 背书 + 排除行 tokenQuality 补句；claude 「已验证集合」节 version→cliVersion 措辞收敛 + `inference_geo` 入映射表标不映射。**已冻结 fixtures 期望值零改动**——逐条核对无任何裁决迫使冻结数值变化（identityQuality 裁决值与已冻结 `strong` 一致；label 裁决值只影响 UNRESOLVED 槽位）。
3. **存量观察 D**：domain.rs 测试非法标识符样本「日志」→「лог」（西里尔，非 ASCII 非 CJK，测试语义不变）。
4. **评审包 §五 已更新**为「已裁决」并逐项记录落地状态与仍开放项（#1/4/9/12/13/14 按既定里程碑挂账）。
5. **遗留对齐项（非本批范围，如实记录）**：契约快照内三处元数据现已过时于修订后 spec——codex 两案的 identityQuality unresolved 声明（spec 行已补，冻结值本就 `strong`）、copilot label 的 `UNRESOLVED:` 标记（裁决已定稿 cwd）、相应 README 措辞。按「fixtures 期望值零改动」指令未动，**建议随 Codex 复审批或各 adapter 接线时清账**（届时 copilot label 按已定稿 spec 冻结为 `/home/dev/proj`）。

完成定义复跑（裁决落地后终态）：fmt / clippy -D warnings / nextest 65/65 / build / deny 四项 ok / xtask ci 全绿；DDL 两项真实执行。收尾批提交已推送同分支。

## 二轮修复（2026-08-04，依 [阶段三B-裁决](阶段三B-裁决.md)「二轮裁决」三项必修；提交 `2259ae5` 已推送）

1. **必修 1（cursor eventKey）**：对照维护者修订后的 cursor.md 单值全键定义逐一核对四个 expected——`bubbleId:<composerId>:<bubbleId>` 全键形式**四处完全一致，零改动**。
2. **必修 2（harness 落地 mask，本批主体）**：`spec_contracts.rs` 重写比较机制——`unresolved` 声明解析为字段路径（`events[<i|*>].<field>`，字段须命中域 serde 面 32 键），比较前从**两侧**摘除该路径，非字符串字段与缺省字段由此真正解冻；**双向断言**：声明解析失败/未知字段/索引越界即红（声明了必被 mask），未声明字段携带 `UNRESOLVED:` marker 即红（与 P6 同结构）。三探针突变自证：标 marker 不声明 → 红；假路径声明 → 红；**被 mask 的数值改动 → 绿**（解冻为真）。四组占位值（copilot ×2 / gemini / opencode 的 tokens、rawSchema、rawUsage）随之解冻，README 承诺同步改写。
   **shutdown rawUsage 二选一**：选**补入 unresolved**，弃「spec 先定 canonical serialization 再冻结」。理由（按"哪个可从 spec 唯一推导"判据）：spec 对 rawUsage 序列化（键序/空白）完全沉默，canonical 形式**不可从 spec 唯一推导**——定它就是替上游发明设计；且裁决自身已把 rawUsage 列入解冻组，reprice 从归一化列重算、无任何消费者需要冻结的序列化。四处 rawUsage 声明措辞统一为「serialization is not a contract surface; content and serialization are pinned at the adapter milestone」。
3. **必修 3（claude 逐字段解析，实现真错）**：四元组 all-or-nothing `Result` 改为按字段独立解析——可解析非负整数保留，仅不可解析维度写 0 并进 `tokenCoverage`，事件仍 excluded + `unverified-schema` 留证（结构信号优先于 cache 拆档裁决；output 不可解析时 reasoning 锚点失效强制 0 以保不变量）。**唯一允许的冻结期望值修正**：msg_K（`input_tokens:"12"`, `output_tokens:3`）→ `tokens_output=3` + `token_coverage=["input"]`；**政策一般化直读**：usage 非对象的退化案（msg_J）四维全不可解析 → coverage 全四维标注（"0 不冒充已确认没有"对其同样成立），一并入快照。断言与 insta 快照同批更新，diff 仅此两事件。
4. 复审小修 `0d60f24`（6 条陈旧 unresolved 清理）已按追认保留为基底。

完成定义复跑（二轮修复后终态）：fmt / clippy -D warnings / **nextest 65/65** / build / deny 四项 ok / xtask ci 全绿；DDL 一致性 junction 挂载真实执行（0 跳过通知），junction 用毕移除。

## 待维护者

1. 分支复审（含裁决落地批 `cfa53da` 与二轮修复批 `2259ae5`）后 squash 合入；
2. 用户签核数据门（三项 + 二轮裁决即 [阶段三B-裁决](阶段三B-裁决.md)，评审包 §五 已同步）→ 阶段四（M1）解锁。
