# 阶段三A · 冻结门独立核验报告

- 查证日期：2026-08-03（三项统一）
- 范围：纯验证任务，与阶段二核心开发零冲突。改动仅三处——ADR-0009「浏览器基线」节回填并修正、ADR-0013 cargo-dist 注记回填、本报告。任务 3 只报告不改动；公开仓零改动。
- 结论速览：
  1. 平台基线：**`Safari >= 17.4` 修正为 `>= 17.5`**（`@starting-style` 门槛实为 17.5），其余各值成立；View Transitions 渐进增强分类经验证必须维持。
  2. cargo-dist：**维护放缓但可用（回退方案待命）**——最新 v0.32.0（2026-05-22），无停维护公告，节奏放缓。
  3. 配置面：骨架**无完全悬空键**；缺失方向两个实质候选（检查更新开关、日志等级）+ 一个前瞻候选；另有三处松散引用与两条附带观察。

## 任务 1：平台基线 caniuse 复核（ADR-0009）

### 数据源

| 来源 | 版本/日期 | 角色 |
|---|---|---|
| MDN browser-compat-data（BCD） | 8.0.8，快照 2026-07-24 | caniuse.com 上 `mdn-*` 页面的底层数据；含浏览器发布日期表 |
| caniuse-db | 1.0.30001806，数据更新 2026-07-16 | caniuse.com 本体数据 |
| WebKit 官方发布文 | 《WebKit Features in Safari 17.5》（2024-05-13） | Safari 修正项的厂商一手佐证 |

两数据集独立下载解析（jsdelivr 分发的 npm 包原始 `data.json`），交叉一致。

### 逐特性核验表（完整支持最低版本，查证 2026-08-03）

| 特性 | Chrome | Edge | Firefox | Safari | 结论 | 来源 |
|---|---|---|---|---|---|---|
| `linear()` 缓动 | 113 | 113 | 112 | 17.2 | 现值正确（门槛低于基线） | <https://caniuse.com/mdn-css_types_easing-function_linear-function> |
| `@starting-style` | 117 | 117 | 129 | **17.5** | **Safari 需改 17.5**——原推定 17.4 错误 | <https://caniuse.com/mdn-css_at-rules_starting-style> · <https://webkit.org/blog/15383/webkit-features-in-safari-17-5/> |
| `transition-behavior: allow-discrete` | 117 | 117 | 129 | 17.4 | 现值正确（Safari 17.4 门槛来自本特性） | <https://caniuse.com/mdn-css_properties_transition-behavior> |
| View Transitions（同文档，渐进增强） | 111 | 111 | 144 | 18.0 | 渐进增强分类必须维持 | <https://caniuse.com/view-transitions> |
| Cascade Layers | 99 | 99 | 97 | 15.4 | 现值正确 | <https://caniuse.com/css-cascade-layers> |
| `:has()` | 105 | 105 | 121 | 15.4 | 现值正确 | <https://caniuse.com/css-has> |
| `color-mix()` | 111 | 111 | 113 | 16.2 | 现值正确 | <https://caniuse.com/mdn-css_types_color_color-mix> |
| 容器查询（size） | 105¹ | 105¹ | 110 | 16.0 | 现值正确 | <https://caniuse.com/css-container-queries> |
| subgrid | 117 | 117 | 71 | 16.0 | 现值正确 | <https://caniuse.com/css-subgrid> |

¹ caniuse 本体把 Chrome/Edge 105 标部分支持（`a`），完整支持自 106；BCD 对 `container-type` 记 105。两值均远低于基线 120，不影响结论。

发布日期（BCD browsers 表）：Safari 17.4 = 2024-03-05、Safari 17.5 = 2024-05-13、Safari 18.0 = 2024-09-16、Firefox 129 = 2024-08-06、Firefox 144 = 2025-10-14、Chrome 117 = 2023-09-12。

### 判定明细

- **Safari：17.4 → 17.5（唯一修正项）**。硬基线特性中门槛最高者是 `@starting-style`（17.5）；WebKit 官方 17.5 发布文将其列为该版新增。原值 17.4 下，「浮层入场退场用 `@starting-style`」这条动效路径对 Safari 17.4 不成立，且该特性无法由 Lightning CSS 降级补齐。原注记「Safari 17.4 与 Firefox 129 是 @starting-style / transition-behavior 的门槛」把两个特性的门槛混为一谈：17.4 只覆盖 `transition-behavior`。
- **Firefox >= 129 成立**：`@starting-style` 与 `transition-behavior` 同在 129 落地（2024-08-06），其余硬基线特性 ≤ 121。
- **Chrome/Edge >= 120 成立（保留）**：硬基线特性实际最高门槛 117（`@starting-style` / `transition-behavior` / subgrid）。120 高于全部门槛，安全；ADR 未记载 120 相对 117 的推导依据，是否收紧到 117 属维护者取舍，本次不改。
- **View Transitions 维持「渐进增强」**：同文档 VT 最低 Firefox 144（2025-10-14 才发布）、Safari 18.0，均高于基线 Firefox 129 / Safari 17.5——基线内确实不保证，WAAPI 回落路径必须保留。跨文档 VT（Chrome 126 / Safari 18.2 / Firefox 未支持）不在特性预算内，仅记档。

### 附带发现（超出本任务改动范围，交维护者）

- [风险登记](../风险登记.md) R7 对冲①（:50）原样复述 browserslist 值，含已过期的 `Safari >= 17.4`——须随 ADR-0009 同步修正；按[文档规范](../文档规范.md) SSOT 原则宜改为链接不复述。R7「遗留动作」（:51，按 caniuse 复核）在本报告后具备关闭条件。
- [路线图](../路线图.md) M0.5 冻结物「browserslist 版本号按 caniuse 逐项复核并回填复核日期」（:34）在本报告后具备勾选条件。

## 任务 2：cargo-dist 存续核验（ADR-0013）

### 事实清单（查证 2026-08-03）

| 事实 | 值 | 来源 |
|---|---|---|
| 最新发布 | **v0.32.0，2026-05-22** | crates.io API 与 GitHub Releases 双源一致 |
| 2026 年发布数 | 3（0.30.4 = 02-17、0.31.0 = 02-23、0.32.0 = 05-22） | <https://crates.io/api/v1/crates/cargo-dist> |
| 仓库状态 | 未归档、未禁用；最近推送 2026-07-28 | <https://api.github.com/repos/axodotdev/cargo-dist> |
| 项目更名 | `dist`（README：“formerly known as cargo-dist”）；仓库与 crate 名不变 | <https://github.com/axodotdev/cargo-dist> |
| 人工维护 | 近 30 个 main 提交中人工 12 个（余为 dependabot / actions bot），最近人工提交 2026-06-03；v0.32.0 由维护者 mistydemeo 发布 | GitHub API commits |
| issue 响应 | 近 30 条已关闭条目中真实 issue 2 条（余为 bot PR）；「v0.32 因依赖被 yank 装不上」的 #2418 当日修复（2026-06-03） | GitHub API issues |
| 积压 | open issues（含 PR）335 | GitHub API |
| 停维护公告 | **未发现**——README、v0.32.0 release 说明、公开搜索均无 | 同上 |

### 判定

三选一结论：**维护放缓但可用（回退方案待命）**。

- 非「不可用」：年内 3 个发布、破坏安装的故障当日响应、仓库未归档、无任何停维护声明。
- 非「活跃可用」：发布节奏较项目早期的月更放缓（0.32.0 后至查证日 10 周无发布）、实质维护集中于单人、近两月 main 提交以 bot 为主、335 条积压。
- 处置：按 ADR-0013 既定条款，`cargo xtask dist` + CI 矩阵自建回退待命，产物形态与安装脚本不变。下次核实点：M5 发布流水线搭建时。

## 任务 3：配置面交叉核查（ADR-0010，只报告不改动）

方法：以 ADR-0010 骨架 17 键 + `machine` 标识为基准，对 internal/docs/ 全部 49 份文档做双向 grep（键名 + 中英文概念词），重点细读组件规格 §8、ADR-0002/0008/0011/0012/0013/0014、风险登记；引用行号经抽样复核（10 处全部命中）。键集合的任何变更属维护者裁决，下表「判读」栏仅为核查建议。

### 方向一：文档引用了但骨架没有的键（缺失候选）

| # | 候选 | 证据 | 判读 |
|---|---|---|---|
| 1 | **检查更新开关**（强） | [ADR-0013](../adr/0013-分发信任链.md):29「**检查更新默认开启**，安装一律由用户确认」；[组件规格](../design/组件规格.md):123「关于」分区含「检查更新」项 | 「默认开启」蕴含用户可关闭且需持久化，骨架无对应键（如 `[updates] check`）。组件规格 §8 同时声明「不引入配置面之外的项」（:115），该项当前无键可投影——两文之一须改 |
| 2 | **日志等级**（中） | [ADR-0014](../adr/0014-诊断与格式演进.md):33「默认 `warn`，`--verbose` 与 `TOKLY_LOG` 提升等级」 | `TOKLY_LOG` 属 `TOKLY_*` 环境变量，而 ADR-0010 优先级链定义环境变量**覆盖配置文件键**——该变量无对应配置键。两解：补日志键，或在 ADR-0010 标注「仅 env/CLI 的诊断项」例外 |
| 3 | **API 源授权存放**（弱·前瞻） | [ADR-0014](../adr/0014-诊断与格式演进.md):37 出站网络两处之一为「用户显式授权的 API 源」 | 授权凭据的存放面在骨架与全部文档中均未定义；属未来源接入面（M3+），冻结时点是否占位由维护者裁决 |

### 方向二：骨架有但无消费者的键（多余候选）

**没有完全悬空的键**。三个键消费关系为隐式/松散，逐条列出：

| 键 | 证据 | 判读 |
|---|---|---|
| `version` | ADR-0010 之外零引用；由 ADR-0010「迁移」节自身消费（单调递增、按版本迁移、失败不覆盖原文件） | 结构键而非功能配置——非多余 |
| `[server] port` | [ADR-0012](../adr/0012-进程与本地服务.md):24「绑定端口 → 生成 token → 原子写 `daemon.json`」消费端口，但未点名 `[server].port` 键；`port = 0`（随机）与端口文件发现机制一致 | 非多余，引用松散 |
| `[quota.<provider>] rule_version_override` | [ADR-0008](../adr/0008-配额窗口.md):59「硬编码规则必须有 `rule_version` 与**配置覆盖口**」、:40「窗口规则……带 `rule_version`、可配置覆盖」 | 非多余，引用松散（两处均未点名键名） |

其余 12 键 + `machine` 消费者充分，摘录如下（行号按 2026-08-03 文档版本）：

| 键 | 消费者（文件:行） |
|---|---|
| `[general] locale` | adr/0007:14 · adr/0001:59 · 架构总览:58 · design/组件规格:120 |
| `[general] timezone` | adr/0011:62 · design/组件规格:120 · 路线图:72 · research/审查-2026-08-03:112 |
| `[general] theme` / `theme_mode` | design/组件规格:120 · 架构总览:58 · adr/0001:59 |
| `[retention] observations_days` | 数据库-schema:70 · adr/0011:29,34 |
| `[retention] events_days` | 数据库-schema:91 · adr/0011:30 · design/组件规格:103 |
| `[pricing] remote_refresh` | design/组件规格:122 |
| `[pricing] override_path` | design/组件规格:122 · adr/0002:176（概念引用）· 路线图:47 |
| `[sources.<name>] enabled` | design/组件规格:119 |
| `[sources.<name>] data_root` | 数据库-schema:58 · design/组件规格:119 · adr/0002:102,120 |
| `[sources.<name>] authoritative_channel` | 风险登记:21 · design/组件规格:119 · 路线图:72 |
| `[quota.<provider>] capacity` | adr/0008:27,29,32,40 · 数据库-schema:352 · research/审查-2026-08-02-二轮:39 |
| `machine` | 数据库-schema:47,57,62 · sources/claude-code:85 · sources/codex-cli:76 · sources/gemini-cli:68 · 架构总览:58 · 路线图:35,49 |

### 附带观察（一并交维护者）

- 组件规格 §8（:115）声明「分区与配置文件段落**一一对应**」，但设置界面无 `[quota]` 与 `[server]` 分区（配额容量/规则覆盖与端口在 UI 无投影处）。「一一对应」与现状不符：或补分区，或把表述改为「设置界面只投影部分段落」。
- ADR-0010 优先级链只写 `TOKLY_*` 通配，全仓无环境变量名录；`TOKLY_LOG` 是文档中唯一具名的 `TOKLY_*` 变量（见方向一 #2）。

## ADR 回填改动摘要

- **ADR-0009**：browserslist `Safari >= 17.4` → **`>= 17.5`**；原推定注记替换为 2026-08-03 复核注记（数据集版本与日期、修正说明、WebKit 一手链接）+ 逐特性最低版本表。
- **ADR-0013**：cargo-dist 注记回填——核实日期 2026-08-03、最新版本 v0.32.0（2026-05-22）、活跃度事实、结论「维护放缓但可用（回退方案待命）」、下次核实点 M5；回退条款原文保留。

## 方法与留痕

数据获取与解析均在本地会话临时目录完成（jsdelivr 分发的两数据集原始 `data.json`、GitHub / crates.io API 原始 JSON），未存入仓库；本报告表格为其全部提炼结论。

