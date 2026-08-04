# 阶段二 · 六源 spec 对齐报告

> 恢复注记（2026-08-04）：原件在 internal/ 误删事故中丢失，本文件由阶段二执行者依会话记录恢复，内容为 2026-08-03 撰写时状态；文末追加了此后已闭合事项的标注。

- 日期：2026-08-03；范围：`sources/` 六份 spec 对照规范 DDL 的六条核对项（文档修正清单「→ 阶段二」节）
- 执行方式：claude-code 由主执行者亲自对齐；其余五份并行派发（Sonnet ×5，各改一文件），全部 diff 经主执行者逐行复审。cursor 的 agent 在最后输出结构化结果时失败，但其编辑已完成且复审合格，无需返工。

## 逐 spec 结果

| spec | ①源标识 | ②projectKey/Label | ③cacheWrite 双档 | ④成本列 | ⑤子表模型 | ⑥两个新节 |
|---|---|---|---|---|---|---|
| claude-code | 已合规 | 已合规 | 已合规 | 已合规 | 已合规 | **新增**（结构性触发条件，见裁决点 2） |
| codex-cli | 常量已是 `codex`；一处「六值枚举」过时旁注改为 ADR-0018 格式约束 | 已合规 | 已合规（无分档→默认 5m） | 已合规 | 已合规（rate_limits 正确走 quota_snapshots） | **新增**；agent 字段未裁决（裁决点 1） |
| opencode | 已合规 | 已合规 | 已合规 | 已合规 | toolCall 映射补了遗漏的 durationMs（`state.time.end - state.time.start`） | **新增**（Gen A/B 迁移名集合；agent=`data.agent`，缺失→''，不回退 mode） |
| gemini-cli | 已合规 | 已合规 | 已合规（无此维度→0+tokenCoverage） | 已合规 | 已合规 | **新增**（结构形态集合 + 未实机验证清单；agent 二值 kind 归一化，裁决点 3） |
| copilot | 已合规（四通道均 `copilot`） | 已合规 | 格式详解 A 的 `cacheWrite` 单值标签改双档 + 恒 0 声明 | 已合规 | 已合规 | **新增**（已验证集合**为空集**——本机无真实样本，诚实声明；agent 四通道恒 ''，并澄清 OTEL "agent turn log" 是记录类型非智能体维度） |
| cursor | 已合规 | 已合规 | 已合规（0+tokenCoverage） | 已合规（estimated 路线 disabled/estimate-only） | 已合规 | **新增**（3.x 三表结构集合；agent 恒 ''，isSubagent 由 parentSessionKey 表达） |

## 待维护者裁决

1. **codex 子代理 agent 字段**（唯一未落定项）：`session_meta.payload.source.subagent.thread_spawn` 下有 `agent_path` 与 `agent_nickname` 两个候选，spec 无取值样本与基数说明——选错高基数字段会破坏 ADR-0018 聚合预算前提。spec 已注明「留待裁定」，归一化映射表未臆造 agent 行。裁定后需回填 spec 两处 + 未来 codex adapter。
2. **claude-code 的「已验证格式版本集合」按结构形态而非 cliVersion 解释**（主执行者裁决，请追认）：该源无独立格式版本字段，若按 ADR-0014 字面把 cliVersion 当 raw_schema 集合，Claude Code 每次发版都会把新用量整批 excluded，与 spec 自身「白名单 + 可选字段兜底、不为特定版本写分支」矛盾。已写入 spec：触发条件是结构性的（usage 非对象 / token 字段非整数），cliVersion 仅记入 `detail_key` 供排查。
3. **gemini-cli 子代理 agent 取 `'subagent'` 常量**（派发 agent 裁决，与 claude-code 的 meta 缺失降级先例一致；上游无更细粒度类型字段）：备选是恒 ''（丢维度）或用子目录名（高基数）。如不认可请指示回改。
4. **claude-code `.meta.json` 缺失时 agent 降级 `'subagent'` 占位**（主执行者裁决）：不用 ''——空串会把子代理用量静默并进主会话桶。

## 遗留

- 六份 spec 的修改均只动 internal 文档，公开仓无痕；对齐后的 spec 是 golden fixtures 的构造依据，M0.5 冻结门的六源 fixtures 按新版 spec 构造。
- copilot 已验证集合为空集是**如实陈述**（调研机无真实样本）——copilot adapter 开工前必须先取样本，否则一切数据按 unverified-schema excluded。

## 后续闭合标注（2026-08-04 恢复时补）

- 裁决点 1 已由维护者裁定并回填：`agent = agent_nickname`（缺失/空 → `'subagent'`；`agent_path` 永不作维度），标注待阶段八实机样本复核。
- 裁决点 2/3/4 已获追认；claude-code payloadHash 后续增补了子代理 meta 掺盐条目（阶段二五轮工作，条目在 spec 映射表内）。
