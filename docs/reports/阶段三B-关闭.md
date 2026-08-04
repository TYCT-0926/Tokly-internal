# 阶段三B · 关闭记录

- 日期：2026-08-04
- 结论：**关闭**。squash 合入 main（`339f727`），分支与 worktree 已清。合并前：rebase 到含阶段五/六的 main（零冲突）→ 全链 **156 测试全过、deny 四项 ok** → PR #4 CI 三平台 + desktop + guard + deny 全 SUCCESS。

## 交付

**六源 golden fixtures**：claude-code 经真实 adapter 接线覆盖 spec 全字段面（cache 拆档裁决、弱键 fallback、unverified-schema 结构触发、subagent 归因与降级）；其余五源以 **spec 契约快照**形式入库（输入 + 按 spec 手工推导的期望 UsageEvent 成对，serde 逐字节往返 harness），标注"待各自 adapter 里程碑接线"。**数据门评审包**（`research/冻结门评审-数据门-2026-08-04.md`）：五件冻结物逐件状态与证据链、spike ① 六项勾稽、18 项未决与 3 组偏差全清单。

## 三轮裁决

| 轮 | 裁决 |
|---|---|
| 一轮（数据门签核三项） | `cache_size` 维持（实测不可区分）；**`WITHOUT ROWID` 采纳 tool_calls + 孪生 event_charges**（插入 1.6×、查询 27%）；spec 收敛十一项；CI 偏差知悉 |
| 二轮（复审 3 必修） | Cursor eventKey **spec 收敛为全键**（通则：映射表任何一格出现两个可选值即契约缺陷）；UNRESOLVED"不冻结"**让门禁兑现承诺**（harness 解析路径 mask + 双向断言），不是删声明；Claude `output_tokens` 整组清零 → **逐字段保留**（本阶段唯一允许改已冻结期望值处，因原值冻结的是错误行为） |
| 三轮（复审停审的契约矛盾） | `claude-code.md` 白名单"其余全部跳过"与 unverified-schema"入库为 excluded"冲突 → **裁 unverified-schema 胜**（源 30 天即删，静默跳过等于永久不可见丢失；且 `:139` 本就写好了边界，是白名单那句没回头更新）。代码零改动 |

## 纪律实证

- **"不猜"红线两次正确执行**：codex paginated 因 spec 自身未定页格式且与审查记载相反而停下（构造即造永久错误基准）；copilot projectLabel 双读法打 `UNRESOLVED:` 不选边。
- **复审停审**：发现契约自相矛盾即停，未继续 mask 突变与冒烟、未冒充通过——这类"两份规格各自成立、合起来矛盾"的缺陷靠跑测试永远发现不了，实现只会选一边然后全绿。
- **终局复审的诚实**：两次外部复核超时无输出，如实写入报告且**未作为通过证据**。
- 六 agent 独立复推对全部冻结数值重新推导，零数值错误。

## 数据门状态

**五件冻结物全部到位**：可执行 DDL 冻结 ✓ · 六源 golden fixtures ✓ · 平台基线与 CSS 特性预算（三A）✓ · 配置面（三A + ADR-0010 增补）✓ · 数据门评审包 ✓。spike ① 验收已随[阶段二-关闭](阶段二-关闭.md)定格。

**待维护者签核三项**（评审包 §五）：① 两项 DDL 裁决点已裁并落码 · ② spec 收敛批次（含 codex paginated 推迟阶段八实机、opencode title 按 ADR 删两处显式冲突的裁法）· ③ CI 偏差知悉（DDL 一致性防公开侧漂移、不防文档侧）。

签核即数据门关闭 → **阶段四（M1 全量）解锁**。
