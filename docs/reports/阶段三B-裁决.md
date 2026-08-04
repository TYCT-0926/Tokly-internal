# 阶段三B · 维护者裁决（数据门签核三项）

- 日期：2026-08-04
- 质量注记："不猜"红线的两次执行（codex paginated 停下、copilot projectLabel 打 UNRESOLVED）判断均正确；六 agent 复推零数值错误是冻结基准可信度的直接证据。

## ① DDL 裁决点（已落 schema 文档，执行者随批落 migration）

- **cache_size -16000：维持**——实测不可区分即无害，超 OS 缓存数据集可能显益，翻案成本为零；打戳完成。
- **WITHOUT ROWID：采纳 `tool_calls` + 孪生 `event_charges`**（同键形/同整集合替换/同按事件查询——结构同一非类推）；`session_model_stats`/`watermarks` 形态与访问不同，未实测不动并注记。

## ② spec 收敛批次（全部措辞级，零已冻结语义改动）

| 项 | 裁决 |
|---|---|
| 2 codex eventKey 双键表述 | 收敛为单值：**内容哈希**（复推确认唯一与规则 4 fork 去重相容的读法）；字节编码细则仍留 UNRESOLVED 待阶段八实机 |
| 3 codex identityQuality | 补行 **strong**：哈希含时间戳形态参数，同刻同参合并的残余风险可接受——行内注明该残余与依据；weak 保留给真正的 fallback 构成键 |
| 5 codex cliVersion | 补一行显式「不映射」（契约已如此冻结） |
| 6 opencode 退化输入 | 采纳 fixtures 读法：**唯一性定义在映射结果**（四假设同结果 = 唯一解 counted）；spec 补退化规则一句 |
| 7 opencode toolCall title | **ADR 为准，spec 删 title**——tool_calls 只存指标，title 是自由文本、贴内容红线；不给 ADR 加列 |
| 8 排除行 tokenQuality | 冻结为 **native**（与 claude 先例一致：excluded 是状态不是数据质量）；三源 spec 各补未命中分支取值 |
| 10 copilot projectLabel | 定稿 **context.cwd 原始值**——label 与 projectKey（=normalize(cwd)）必须同源，否则展示与聚合精神分裂；repository 不映射（如有价值属未来 detail，另议） |
| 11 copilot periodEnd | 显式化为 **shutdown 时刻**（覆盖期终点读法，与 month/day 同构） |
| 15 gemini 示例数字 | 修正为自洽值（13046，与 fixtures 一致） |
| 16 gemini 折叠留存背书 | 采纳，spec 补一句 |
| 17/18 claude 措辞 | 以映射表为准收敛：version→cliVersion、rawSchema=None 现状如实；inference_geo 入映射表标「不映射（M1 顺带评估存档）」 |

## ③ CI 偏差

知悉：DDL 一致性测试防"公开侧漂移进 PR"，不防"文档侧漂移"——后者靠本地门禁与维护纪律（文档规范已登记）。

## 存量观察 D

domain.rs 测试里的 CJK 字面量换任意非 ASCII 非 CJK 样本（降信号零成本），随本批。

## 归属

①-b migration/一致性测试/快照 + ② 全部 spec 措辞 + D → 三B 执行者同分支收尾批；随后 Codex 复审全分支 → squash 合入 → **用户签核数据门**（三项即本文件）→ 阶段四解锁。

---

## 二轮裁决（2026-08-04，复审"不通过"三项）

三项全部成立且判据准确（"golden 不得代替 spec 做设计决定"是本轮明确判据）。裁决如下——**其中两项根源在 spec 或 harness，一项是实现真错**：

### 必修 1 · Cursor eventKey 双读 → **spec 收敛为全键**（已改）

`cursor.md` 映射表同格给两个可选值，本身就是契约缺陷。定为 **`bubbleId:<composerId>:<bubbleId>`**：`bubbleId` 全局唯一是上游未承诺的观察，被推翻即静默跨会话撞键，而事件身份入库不可逆；全键的代价只是键长。fixtures 现值恰与裁决一致 → **四个 expected 零改动**，但这不改变判据——复审拒绝"fixture 自选"的立场正确，外部 Claude 的反对意见不采纳。
**通则入约**：映射表任何一格出现两个可选值即视为契约缺陷，实现前必须收敛（写入 spec 编写规范）。

### 必修 2 · UNRESOLVED 的"不冻结"未被 harness 实现 → **harness 落地 mask**（不是删声明）

README 承诺与门禁不一致，是**验证机构缺陷**（与阶段二五轮同类）。裁决取"让门禁实现承诺"而非"改承诺就范"：
- harness **解析 `unresolved` 里的字段路径，在比较时 mask 该路径**（非字符串字段、缺省字段因此都能被解冻）；
- **声明与实际一一对应**：声明了却未 mask、或标了 marker 却无声明，均判红（双向断言，同 P6 结构）；
- `rawUsage` 的键序/空白**不是契约面**——`session-state-shutdown` 那条按此补入 `unresolved`，或由 spec 先定 canonical serialization 再冻结，二选一由执行者按"哪个可从 spec 唯一推导"决定，选后者须 spec 先行。
- 四组占位零值（copilot ×2 / gemini / opencode 的 tokens、rawSchema、rawUsage）随之解冻。

### 必修 3 · Claude `output_tokens=3` 被整组清零 → **实现改为逐字段保留**（改已冻结值）

实现与 ADR-0014「按可解析部分入库」及 claude spec 明文相反——四元组 `Result` 一错全零是实现取巧，不是契约。按字段独立解析：可解析的非负整数保留（本例 `tokens.output=3`），不可解析维度写 0 并进 `tokenCoverage` 标注（"0 不冒充已确认没有"，ADR-0002 既定），事件仍 `excluded` + 留证。**同批更新测试断言与快照**——这是本阶段唯一一处允许改已冻结期望值的修改，因为原值冻结的是错误行为。复审"仅反馈不代改"的处置正确。

---

## 三轮裁决（2026-08-04，复审停审的契约自相矛盾）

复审在 `claude-code.md` 发现 `:24`（白名单：usage 非对象 → 其余全部跳过）与 `:139`（unverified-schema：usage 非对象 → 入库为 excluded + 留证）**直接冲突，契约无法唯一执行**，遂按「文档与代码分歧是最高级事故」停审，未继续 mask 突变与冒烟、未冒充通过——**处置完全正确，这正是该纪律存在的意义**。

**裁决：`:139` 的 unverified-schema 语义胜，`:24` 白名单原则改写**（已落 spec）。理由三条：

1. **时序与意图**：白名单原则写在 ADR-0014 之前，目的是"不靠黑名单枚举类型"；unverified-schema 是后补的**格式演进可见性**机制，目的完全不同且更晚经过审议。
2. **fail-visible 是本项目的硬纪律**：源 30 天即删，**静默跳过等于永久不可见的丢失**；"usage 存在但形状变了"正是格式演进的典型信号，少算必须可见（ADR-0014「少算是可见的，错算是不可见的」）。
3. **`:139` 本就写好了边界**：它已明确"usage 完全缺失的 assistant 行按既有白名单跳过，不属格式演进信号"——说明它是带着白名单一起想的，是白名单那句没回头更新。

**分界现已明文**：非 assistant 行、或 assistant 但 usage **完全缺失** → 跳过不留证；usage **存在但非对象**、或已知 token 字段非整数 → 入库 excluded + 留证。

**实现与测试无需改动**（它们选的就是 `:139` 语义，二轮必修 3 的逐字段保留也建立在此之上）；复审可直接从停审点续跑。
