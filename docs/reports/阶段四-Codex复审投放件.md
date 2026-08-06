# 阶段四 M1 · 跨模型对抗复审投放件（交 Codex）

- 日期：2026-08-06
- 用途：M1 合入前的**最后一道复审**，纠正此前四轮均由 Claude 侧 agent 复审的破例（同族互验 ≠ 独立验证，见 AGENTS.md 派发协议）
- 投放方式：`codex exec -c model_reasoning_effort="max"`（触发 max 档的高危条件：资金 micro-USD 计价 / 不可逆数据回收 / 跨持久化边界，三条同时命中）
- 待审范围：`feat/m1-data` 的 `b18d521..3df55fe`（21 提交），worktree `E:\Axiom\Tokly-m1`

---

## 投放正文（以下整段发给 Codex）

```text
# 跨模型对抗复审：Tokly M1 数据层最终把关

你是独立复审方。此前四轮复审全部由 Claude 侧 agent 执行——同族互验不等于
独立验证，你的价值在于**不共享它们的盲区**。目标不是确认它能用，是证明它错。

## 环境
- 仓库 E:\Axiom\Tokly，待审 worktree `E:\Axiom\Tokly-m1`（分支 feat/m1-data，
  HEAD 3df55fe）。范围 `b18d521..3df55fe`，21 个提交。
- `E:\Axiom\Tokly\internal\` 是私有文档仓：**只读**。
- **绝对禁止删除任何 worktree 或目录**——本项目已两次因 junction 跟随而丢失
  internal 工作副本。探针请建在你自己的临时目录，以路径依赖引用被审 crate。
- 你只读、不改、不提交、不合并。收尾请确认被审树 `git status --porcelain` 为空。

## 必读（判定权威，按序）
1. internal/docs/adr/0002-数据架构.md —— **全文**，尤其文末 2026-08-05 的
   定案 1–7、证据字段补文、点 6 规格补文、观测层统一律、FPR 裁决更正。
   这是唯一契约权威；实现与它任何不一致都是发现。
2. internal/docs/adr/0011-存储生命周期.md（四层寿命、永久层预算、rollup 失效
   缺口记录）、0015-计价精度.md（skipped 三因）、0019-数据新鲜度.md（谓词域
   = fold ∪ health；发布端分期至 M2）、0014-诊断.md。
3. internal/docs/数据库-schema.md（规范 DDL，含 v5 的 Bloom 判别器表）。
4. internal/docs/reports/阶段四-复审.md（一轮全量复审 16 项）与
   internal/docs/reports/阶段四-M1数据层.md（执行者报告，1088 行，含四轮修复）。
5. internal/docs/开发流程.md 测试策略节（P5/P6 构造规则、proptest 种子入库、
   门禁覆盖律）。

## 已知既有失败，不要报
- `tokly-service::service_queries::the_generation_and_the_numbers_come_from_one_commit`
  —— 阶段七既有的负载敏感 flake。
- 桌面树 `cargo deny` 的 gtk-rs unmaintained advisories —— 既有基线，已挂账 M5。

## 重点攻击面

### A. 定案 7 的判别器（本批最高危）
分片可扩展 Bloom，按 `(source_instance_id, event_key)` 键控，替代了被三轮证伪
的时间水位。逐条打：
1. **无假阴性**：读 `crates/tokly-core/src/reclaimed.rs` 全文，找任何清位、
   重建、截断、丢片的路径（容量增长的链接续、序列化往返、进程重启、多次
   compact 合并、并发写）。任一丢位 = 假阴性 = 永久双计。
2. **判据是身份单项**：四轮曾抓到判据里混入 `projection_absent`（非剪枝
   不变量），导致重建腿静默丢数。确认现在判据不含任何随重建/剪枝移动的
   合取项，且被拒键的投影行确实并入了「重建清除例外类」。
3. **两腿一致**：亲自构造增量腿 vs 重建腿的终态比对，**不要采信"由构造
   成立"**——这句话在本裁决上已被撤回两次。实现里有运行期自检，试着让它
   误报或漏报。
4. **假阳性可见性**：强制小容量制造真实假阳性，确认留 `reclaimed-key-reobserved`
   证据且字段按补文（两列同源、缺一置 NULL、聚合取成对而非各自最小）。
5. **FPR**：执行者称逐下标独立雪崩后实测 1/36,364（在 2⁻¹⁵ 内）。独立复测。
   注意四轮的教训：片宽取 2 的幂反而更差（1/2,614），根因是等差数列相关。

### B. 观测层统一律（最后一轮新立）
一切零触碰类（pruned / baseline / reclaimed）**只跳过投影层与永久层，不跳过
观测层**：到达一律入观测层、参与同坐标裁定（normalizer 升级替换、divergence、
规则 3/4 冲突派生）、按补文落证据。
- 找任何仍然短路的分支；
- `IngestOutcome` 是否真能区分「投影跳过但观测已处理」与「完全跳过」；
- `data_version` 是否按 ADR-0019 谓词判（裸观测追加不递增；产出证据行属
  health 域递增）——构造两侧反例。

### C. 收敛契约四场景 + P5/P6
- 四场景（交换/乱序/重放/重建）在批事务 + Bloom + 观测层统一律三者叠加后
  是否仍成立；
- **P5/P6 自身**是本项目最高危区：oracle 不得询问 SUT、模型只准从 ADR 条文
  直译（类封死三条）。四轮抓到过「模型用精确集合替换概率结构，使判别器唯一
  的误差方向落在模型域外」——检查同类抽象盲区是否还有；
- `crates/tokly-core/tests/*.proptest-regressions` 刚从 gitignore 里救回来
  （此前 CI 从不重放）。确认种子确实被跑到，且其中的反例仍红→绿可复现。

### D. 资金不可逆面
i128 累加、单次 round-half-even、price_kind 字典序求和、priced-at-snapshot、
skipped 三因（第三因判别项是 `frozen_baseline_at` 非空，不是 `has_partial_cost`）、
reprice 幂等 + 推进 data_version + 置 dirty、pruned/frozen 双守卫。
一轮曾抓到 pruned 半边**完全没有回归**（删守卫 494 测试全绿），复查其杀伤力。

### E. compact 与永久层
四入口不可变（pruned/frozen）、按键整片重算含删除缺席非 frozen 键、尾账在
删除同事务内累加、冻结与计价解耦（回收恒可推进）、冻结时 pending 基线落章
skipped 且金额/版本 NULL。构造它们之间的缝隙。

### F. 报表口径
四条腿（非冻结汇总 / 冻结定格汇总 / 尾账 / 存活尾侧）+ 无会话事件。验收 =
reactivation 情形下报表与事件级真值**逐字段相等**。找双计或漏计的组合
（冻结会话 × 尾侧 × 部分成本 × 跨天 × 复活）。

### G. 门禁与机构本身
- 亲跑：`cargo run -p xtask -- ci`（现已含桌面树三步）、
  `cargo nextest run --locked --workspace --all-features`、`cargo deny check`、
  `cargo run -p xtask -- mutate`；
- **mutate 的裁判可信度**：等价声明须用全量套件验证（四轮抓到过假等价，
  也抓到过编译失败冒充 killed）。逐个检查 2 个等价声明与 36 个 mutant 的
  锚点唯一性与还原；
- 门禁是否真覆盖它声称覆盖的（本批刚修了桌面树漏审；找同类漏洞）。

## 已知未决，不要重复报（已裁决入约）
- 首次导入 SLO 32.2s vs 30s —— 已裁转阶段八验收（当前基准六源里五源无
  adapter、83% 事件跳过解析，不具代表性）。
- 实例身份稳定性（adapter 学到 account 换 id → 历史重计）—— 已立案路由
  阶段八，M1 内只做到 doctor 可见。
- rollup 失效追踪 —— 已记为 M2 前置（该表当前零消费者）。
- ADR-0019 的 health 发布端无写入者 —— 有意分期至 M2。
- tokly-pricing 疑似 flake（约 30 次超订跑 1 次，不可按需复现）—— 观察项。

## 输出
结论：**通过 / 通过附建议 / 不通过**（不通过须列必须修复项，每项给可复现
反例与命令）。逐项核验表，每项给你实际运行的命令与结果。发现分级：
必须修复（契约违背 / 数据丢失 / 资金错算）vs 建议。
报告写 internal/docs/reports/阶段四-Codex复审.md。**不合并。**
```

---

## 投放后的处置

- **通过 / 通过附建议** → 编排者裁决遗留项 → squash 合入 main → 写阶段关闭记录 → 解锁阶段八与七B；
- **不通过** → 必须修复项发回执行者，修完只复审修复点。

## 本投放件的由来

AGENTS.md 派发协议写明「执行 = Claude，复审 = Codex；高危阶段对抗式」。M1 前四轮复审全部由 Claude 侧 agent 执行，虽各自抓到真缺陷（含两次推翻维护者裁决），但**同族互验存在共同盲区**——四轮那条「模型用精确集合替换概率结构，使判别器的误差方向落在模型域外」正是此类盲区的实例。本轮跨到 GPT 侧收口。
