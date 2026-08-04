# 阶段九 · TUI 实时监控与历史浏览 执行报告

- 日期：2026-08-04
- 分支：`feat/tui-slice`（git worktree `E:\Axiom\Tokly-tui-slice`，基于 main `b660db6`；与 feat/freeze-closure、feat/web-slice 零文件交叠）
- 结论：**完成定义全绿**。监控 + 历史双视图、令牌降级链三档真机像素级命中、10fps 节流、80/60 列降级、braille 热力图密度映射、ADR-0019 新鲜度契约读侧全量落地。212 测试全过（新增 66），fmt / clippy -D warnings / nextest / build / deny 全绿。
- 待裁决一项：**core 侧 `meta` 表迁移未落**（见「新鲜度落地 · 缺口上报」）。

## 交付物

| 项 | 位置 |
|---|---|
| TUI 主体（双视图 + 诊断视图、主题、节流、降级） | `crates/tokly-tui`（src 12 模块 + build.rs 令牌内嵌） |
| 格式化规则库（ADR-0016 两 locale，零依赖，不引 ICU） | `crates/tokly-format`（含跨语言测试向量 `tests/vectors.json`） |
| service 查询面扩展：模型/项目/最近事件 + **复合单事务 overview** + data_version | `crates/tokly-service`（query/lib/dto） |
| ADR-0019 契约 DTO（Overview / ChangeEvent / ChangeScope，ts-rs 入库） | `crates/tokly-service/bindings/`（新增 6 个 .ts，固化测试覆盖） |
| 富种子（精确期望）+ 年度种子（视觉/冒烟） | `test_support::seed_rich_db` / `seed_demo_year_db`（纯增量，既有种子未动） |
| TestBackend 三宽度快照 ×14 | `crates/tokly-tui/tests/snapshots/render_snapshot__*.snap` |
| 截图矩阵 40 张 SVG + 动画录屏 + 真机 ConPTY 三档 PNG | `internal/docs/reports/assets/九/`（**双击 `index.html` 即览，零安装**） |
| 截图/种子生成器（dev-only） | `crates/tokly-tui/examples/{shots,seed_demo}.rs` |

## 完成定义核对（全绿）

- `cargo run -p xtask -- ci`（fmt --check / clippy --locked -D warnings / nextest / build，全 workspace 含新 crate）：**exit 0**
- `cargo nextest run --workspace --all-features`：**212 过 0 败**（1 例跳过为 core 既有路径；本阶段新增 66：format 3 · service 8 · tui 55）
- `cargo deny check`：四项 **ok**，零新依赖（tokly-format 零依赖；tui 新增仅 workspace 内部 crate 与既有 serde_json/time）
- 分支已推送并开 PR（见文末）；不 self-merge

## Windows Terminal 兼容性冒烟（ConPTY，路线图 M4 前置）

真机（Windows 11 · Windows Terminal 默认终端接管 · release 构建 · `seed_demo_year_db` 种子库）三档 `--color` 实测，窗口捕获后**逐像素采样比对**：

| 档 | 实测采样 | 期望 | 判定 |
|---|---|---|---|
| truecolor | R224 G152 B90 | `accent.solid` **#E0985A** | **逐字节命中** |
| 256 | R215 G135 B95 | xterm 索引 173 = **#D7875F** | **逐字节命中** |
| 16 | 视区零饱和像素 | amber accent 语义表落 `white`，色相丢失 | **与令牌系统「已知损失」条目一致** |

三档窗口捕获哈希互异；`conpty-*.png` 存 assets。字节层另有自动化证据：`tests/ansi_stream.rs` 对真实 CrosstermBackend 字节流断言——truecolor 含 `38;2;`、256 仅 `38;5;n`（含 n≥16）、16 档索引全部 <16。

**渲染宽度**：边框、右对齐数字列、braille 字符（U+2800 系）、状态字符 `✓ ✗ ! ○ ◐ ●` 在 ConPTY 下全部单宽对齐（见截图，边框零错位即是判据）；全程无 emoji。截断符 `…`（U+2026，East Asian Ambiguous）在 WT 实测单宽，正常。

**冒烟捕获的一个环境发现**：本机自动化环境带 `NO_COLOR=1`，首轮捕获全灰。排查定位到 crossterm 内建 no-color.org 支持（色码静默置空、其余序列照常）——即 **TUI 天然遵守 NO_COLOR 规范**，属正确行为，记录备查。清除变量后重拍得上表结果。

**交互实测**：`q` 干净退出（多次验证，exit 0）、`t` 主题切换、`Tab` 视图切换在真机送键验证通过。交互过程截图因维护者桌面活跃（Windows 前台锁保护）未能全套捕获，由 `session.svg` 动画（同一渲染代码逐帧生成）承担：启动骨架 → 数据到达 → t×2 → Tab 历史 → 80 列 → 60 列 → 摘要行。

## 三宽度 TestBackend 快照

`tests/render_snapshot.rs`，快照在 `crates/tokly-tui/tests/snapshots/`：

- `monitor_120 / monitor_80 / monitor_60`、`history_120 / history_80 / history_60`——两视图三宽度；
- `monitor_59_is_the_single_line_summary`——低于 60 列的单行摘要模式；
- `diagnostics_120_shows_the_degraded_mode`、`loading_placeholders_share_the_ready_shape`、`not_initialized_is_an_instruction_not_an_error`、`newer_schema_refuses_with_versions` 等状态快照。

80 列全列在场；60 列 recent 表丢 output/cache 两列、维度表丢 cache（**整列丢弃，无压缩无数字截断**，`ui/table.rs` 的 `visible_count` 单测钉死各断点）。

## 重绘节流实测

- 机制：`throttle.rs` 独立模块——`mark_dirty / should_draw / drawn / until_due` 四原语；ratatui 双缓冲 diff 保证单帧只 flush 变更单元格。
- 实测（单测 `a_continuous_dirty_stream_is_capped_at_ten_fps`）：**模拟 1kHz 连续脏源 2 秒 → 恰 20 帧**（=10fps 上限）；窗口内多次 mark 合并单帧（`marks_inside_one_window_merge_into_one_frame`）。
- **无变化零重绘**：干净屏 `until_due = None`（事件循环不为渲染醒来）+ 数据层 `PartialEq` 比对 + 版本门控（版本未动连查询都不发）。三层叠加后，空闲监控的稳态帧率为 **0**。
- 运行期可观测：诊断视图（`d`）常驻 `render N frames in 10s · x.x/s · cap 10/s` 读数，维护者可实机复核。
- 输入风暴防护：事件循环把队列内全部输入合并进单帧（按键连发 = 一次重绘）。

## 降级路径验证

| 路径 | 触发 | 表现 | 验证 |
|---|---|---|---|
| 只读直连 | 无 daemon 持锁（探锁即得） | 状态栏常驻 `◐ read-only · updated hh:mm`（warning 色 + 时钟） | `read_only_marks_the_status_bar` 快照断言；`data_fetch` 集成测试 |
| schema 高于本体 | `schema_migrations` 版本比对（Service::open，直连降级同判据，ADR-0012） | 拒开 + `store v9 · supported v1` + 升级指引，**不做尽力解析** | `newer_schema_refuses_with_versions` |
| 库缺失 | NotInitialized | `○` + 一句指引 + 一个动作（scan），**非错误态语气** | 快照 + 集成测试 |
| 重取失败（已有数据） | 任意 fetch 错误 | **旧值保留** + `! stale` 标注，细节入诊断视图 | `a_failed_refetch_keeps_old_data_and_marks_stale` |
| 探锁失败 | io 错误 | `! probe failed` | 状态字符三态映射（●/◐/!，与 Web 状态点同语义词汇） |
| 库中途出现/消失 | 运行中 | Fetcher 丢句柄重开重握手，恢复即续 | `a_failed_poll_recovers_once_the_store_exists` |

## 新鲜度落地（ADR-0019）

**版本传播链路**（本阶段 TUI 恒为只读直连形态——ADR §6 指定的轮询例外路径）：

```
写侧提交（未来 M2 摄取）→ meta.data_version+1
  → TUI 每 5s 轻探 Service::data_version()（单行读）
  → needs_refetch(last_seen, current)？
      否 → 零查询零重绘（仅重探 daemon presence）
      是 → Service::overview()：单只读事务（unchecked_transaction）内
           算完 summary/daily/models/projects/recent + 读 data_version
           → 整屏替换，重绘一帧
  → r 键 = invalidate()（弃基线）+ 立即 poll
```

- **复合视图单版本**：`Overview` DTO 一次事务产出全部块（WAL 快照隔离），「总量对不上明细」在构造上不可能；TUI 不再拼装原子查询（旧的 5 查询拼屏写法已在本阶段内废除）。
- **漏事件恢复**：`needs_refetch` 纯函数——版本相等是唯一免取条件；+1、跳代（gap）、倒退（库被替换）、任一侧无版本，一律全量重取不补差。单测 `refetch_gate_follows_the_adr_0019_client_rule` 七分支全钉；集成测试 `version_gate_skips_refetch_until_the_store_moves` 验证 7→12 跳代路径实取。
- **三态表现**：新鲜 = 正常；重取中 = **旧值保持 + 300ms 延迟 braille spinner**（本地查询常在个位毫秒完成，无延迟的 spinner 会把安静角落变成频闪——无骨架屏无闪烁，骨架only首载）；陈旧 = 状态栏常驻 `◐ read-only · updated hh:mm`（读侧确认时钟，分钟粒度），重取失败叠加 `! stale`。触发与截图见降级表与 assets。
- **事件结构入契约面**：`ChangeEvent { dataVersion, scopes }`、`ChangeScope`（events/sessions/rollups/quota/health）已入 ts-rs 生成面（七A mock 可直接消费）；广播的发布端随 M2 摄取与 SSE 升级接线，TUI 获得进程内写者后切换到订阅路径（当前形态无进程内写者，按 ADR 例外走轮询）。

**缺口上报（按红线停下上报，不自行择一）**：ADR-0019 与 schema 文档均已含 `meta` 单行表，但 **main 上 `tokly-core` 的迁移（v1）尚未创建它，写路径也未实现同事务 +1**——该表归 core 轨道（feat/freeze-closure 在辖）。本阶段读侧的处置：`data_version` 读取对缺表宽容——返回 `Option::None` 表示「库先于新鲜度契约」，**不伪造版本 0**；无版本 → 无法门控 → 每次轮询全量重取（等价于旧行为，语义诚实）。core 落迁移后版本自动流通，TUI 与 service 零改动。`Overview.data_version` 因此暂为 `number | null`；若维护者裁定 core 侧本批同落，可收紧为非空。测试两态均覆盖（缺表 → None；手工建表 → 版本流通）。

## TUI 完成度清单（§13 逐项）

| 项 | 结果 |
|---|---|
| 焦点态可见 | 焦点面板边框 accent、非焦点 fg.faint；监控视图唯一可滚面板恒焦点。`render_style.rs` 按单元格样式断言（含焦点随 ←/→ 移动） |
| 全键盘可达 | q/Esc · t · r · d · Tab · ←→(hl) · ↑↓(jk) · PgUp/PgDn/Home/End；完整键表常驻诊断视图；滚动钳位（`scrolling_clamps_to_data`）；Windows Release 键过滤保留 |
| 无数据/加载/错误三态 | 四态实现：库缺失（指引）、零事件（`run tokly scan` 一句话+动作）、加载（等形占位，CLS=0）、错误（`!` + 细节 + `r retries`）；快照各一 |
| 降级态明示 | `◐ read-only` + 时钟常驻状态栏；`! stale`；诊断视图全细节 |
| 主题切换保持位置 | `t` 只动调色板；`theme_switch_preserves_view_focus_and_scroll` 钉死 view/focus/scroll 三不变 |
| TestBackend 三宽度快照 | 120/80/60 双视图 + 59 摘要模式 + 状态快照，共 14 |

## 关键设计裁决（含被弃备选）

1. **官方主题构建期内嵌**（build.rs 以 tokly-tokens 为 build-dependency，派生后以 TUI 产物 JSON 形态嵌入二进制，运行时 serde 解析）。与 `target/tokens/tui/` 产物内容逐字节同源（同一引擎同一源）。备选「运行时读 target/ 目录」弃：安装态二进制无 target；「运行时派生」弃：违背运行期零计算。同一解析器将来直接服务 `tokly theme install` 的用户主题 JSON。
2. **版本轮询取代定时全量刷新**（原 2s 全量 → 5s 单行版本探测 + 变更才复合重取）。ADR-0019 追加后当日重构；空闲成本从每 2s 五查询降到每 5s 一查询、零重绘。
3. **重取失败保旧值**是 ADR-0019 与既有实现的唯一真冲突（旧实现整屏转错误态），按 ADR 改：有数据时错误只以 `! stale` 标注呈现。
4. **spinner 延迟 300ms**：毫秒级刷新不配 spinner，闪烁比无反馈更糟；超 300ms 才出现（细微进行中指示），首载骨架不受影响。
5. **today 面板口径纯化**：初版第二行放了全库 sessions，真机冒烟审图时发现与面板「today」标题口径矛盾，改为当日 output——面板内一个口径。
6. **表格自绘**（不用 ratatui Table widget）：列宽/对齐/整列丢弃语义全部显式，降级规则可单测。
7. **fixture 伪随机换 splitmix64 终结器**：线性同余走格在热力图上产生对角纹（签名对象读起来像摩尔纹不像生活），换混淆函数并加周末低量、假期空档、渐进增长塑形。
8. **诊断视图承载全键表与运行细节**：状态栏保持规格四键提示的安静，发现性下沉到 `d`——仪表盘的控制说明印在门盖内侧，不贴满面板。
9. **duration/时刻格式**：`3.2s / 1m12s / 2h14m`（§2 重置时钟同形）；recent 表时间列当日 `HH:MM:SS`、跨日 `MM-DD`（单一规则按日界切换，可预测）。全 UTC，时区子系统（ADR-0010 设置面）落地前不做本地时区，诊断视图明示 `all timestamps utc`。
10. **tokly-format 严格按 ADR-0016 表实现**：紧凑记数定为「3 有效数字 + 尾零裁剪」（规格例 3.4M/12.4K/1.84M/412K 全部吻合此律）；duration 不在 ADR 表内故留在 TUI 侧。跨语言向量 fixture 置于 crate 内，TS 侧接线位置待维护者定。

## 设计自查（反模板设计守则 §五）

**这一屏在模仿什么真实物件**：一台放在机架上的通道监视仪——示波器的读数区 + 电平表的刻度 + 航电的状态灯。数字是主角，色彩只指示状态与焦点，安静是默认态。三处体现：①状态栏是仪器的测量条——模型名（通道名）用唯一一处 accent，读数右对齐等宽，键位提示像面板丝印一样退到 fg.faint；②trend 火花线带右缘刻度（max/0），像示波器格线上的电平标尺，而不是装饰性缩略图；③空闲稳态零重绘 + 版本门控——仪器没有新读数时指针不抖。

**八条 AI 味症状逐项**：

| # | 症状 | 自查 |
|---|---|---|
| 1 | 均匀圆角卡片阵 | 不命中。终端无圆角；面板边界仅发丝级单线框，焦点面板才有色彩边界，today/trend/热力图恒 faint——层级不均匀是刻意的 |
| 2 | 标题+图标+副标题三件套 | 不命中。面板标题嵌边框单行（`─ today · Aug 4, 2026 ─`），无图标无副标题；副信息（窗口、滚动范围）以 faint 注记并入标题行 |
| 3 | 等分栅格 | 监控视图按信息权重分配：today 4 行、trend 4 行、事件流吃掉其余全部（它是"实时"之所在）。历史视图两表在 120 列并排属对称例外——两个维度权重确实相等 |
| 4 | 色彩做分类 | 不命中。单 accent + 灰阶；模型/项目靠位置与排序区分；仅 status.✻ 三色做小面积状态 |
| 5 | 动效均匀涂抹 | 不命中。全部动效 = 一个 300ms 延迟 braille spinner；视图切换无转场；主题切换同帧换色 |
| 6 | 数字当文本排版 | 不命中。一切读数右对齐 + 等宽（终端天然）；成本固定两位小数成列；紧凑记数 3 有效数字 |
| 7 | 空态插画+鼓励语 | 不命中。`no events yet. run tokly scan to import your first source.`——一句指引一个动作 |
| 8 | 首屏 hero | 不命中。首屏第一行就是 today 读数；产品名只出现在 <60 列摘要行的行首（那里它是数据的主语） |

**十项可验收细节（按 TUI 形态落实）**：

1. 光学对齐：状态字符后置单空格再接文字（`◐ read-only`），braille bar 与数字列基线同行——单元格制下光学=数学，锚点在字符选择（全部单宽）。
2. 小数点成列：成本恒两位小数 + 右对齐 → 小数点天然成列（快照可验，`$18.20/$16.85/$0.67`）。紧凑 token 记数（12.4K/1.84M）右对齐但小数点不成列——混合数量级的紧凑记数无法两全，列入主观存疑 ①。
3. hover 不位移：TUI 无 hover；对应物是焦点切换只改边框颜色，零布局位移（快照对比可证）。
4. 焦点环不裁切：焦点即面板边框本身，不存在被裁切的外环。
5. 首屏无抖动：加载占位与真实读数等宽等位（`–` 右对齐在同一字段），数据到达零位移；`loading_placeholders_share_the_ready_shape` 快照钉形。
6. 发丝线：终端单线框即最细线；焦点/非焦点以色阶（accent/faint）而非线宽区分。
7. 长文本策略：模型名尾截断（前缀载识别信息）、路径头截断（叶目录载识别信息），各自唯一规则（`truncation_is_end_specific` 钉死）。
8. 零与缺失可区分：成本 `-` ≠ `$0.00`（同屏可见：07-31 行 `-` 与 08-01 行 `$0.67`）；库缺失 ≠ 零事件（两套文案两种语气）；未来日历格空白 ≠ heat0；无版本 ≠ 版本 0。
9. 键盘完整回路：见完成度清单；Esc 关诊断回原视图、d 二次切回原视图（焦点归位的 TUI 对应物）。
10. 两主题各自成立：浅色不是反色——令牌引擎按 light 独立派生（heat 反向强度、前景独立 L 阶）；`amber-light`/`paper-light` 截图在 assets 单列。TUI 侧用法层面深浅共用同一语义槽位，成立性由令牌保证。

**主观存疑（数值合规但观感存疑，交维护者裁决）**：

1. 紧凑记数混合数量级（58K 与 1.99M 同列）右对齐后视觉重心略跳。备选是固定单位（全 K 或全 M），但会产生 0.06M 这类退化读数。我判断现状更优，存疑备案。
2. 120 列下热力图 53 格居中于 118 格面板，两侧留白约 30 格。曾考虑并排放置图例或年度汇总，因规格未定义此组件而作罢（不即兴发挥）。留白是安静的，但"是否浪费了一带"值得维护者看一眼实图。
3. 16 色档 amber 全灰（accent 落 white）在监控视图几乎无色（仅状态点绿/黄/红）。这是语义表的既定结果，但观感上"16 色档不如 mono 主题有意"。若在意，唯一出路是修 ANSI16 语义表（令牌侧决策，非 TUI 侧）。
4. 历史视图 120 列两表并排时各半宽 56 列，cache 列与 mini bar 双双放弃；80 列堆叠反而全列 + bar 齐全——"更宽的终端看到更少的列"有悖直觉。备选是 120 列也堆叠（视觉上矮胖）。我按"并排两维度并列可比"优先，存疑备案。

**维护者验收指引**：双击 `internal/docs/reports/assets/九/index.html`——首屏动画即交互全流程；三张 conpty PNG 是真机三档；矩阵段看 60 列降级与 16 档密度热力。实机跑：`cargo run -p tokly-tui --example seed_demo -- --db target/demo/tokly.db && cargo run -p tokly-tui --release -- --db target/demo/tokly.db`，依次按 `t`（主题十连循环）、`Tab`（历史）、`d`（诊断，看 fps 读数与键表）、缩窗到 60/59 列（列丢弃与摘要模式）。

## 与规格的偏差与上报事项

1. **core `meta` 迁移缺口**（见新鲜度节）——唯一阻塞版本门控实际生效的外部依赖；读侧已就位。
2. 原子查询 DTO（summary/daily）顶层未加 data_version——ADR-0019 字面要求"每个查询 DTO"；本批只在复合 Overview 落，原子 DTO 改动牵动 tokly-server 与桌面（他轨），建议随 SSE 升级批一并做。
3. i18n：文案全部 key 形态（`msgs.rs`，Fluent 接线位已留、`set_use_isolating(false)` 要求已注释在接线点），本阶段英文常量——任务书允许。§12 的 zh 80 列布局校验随 i18n 接线补。
4. 状态栏在 TUI 位于底部（§13 规定），ADR-0019 的"顶栏陈旧标注"按 TUI 形态映射到底部状态栏——语义等价（常驻、明示、含时钟）。
5. 配额油表（§2）不在本批：quota_snapshots 无数据源（M3 前置），braille 条形态已按 §2 落在表格 mini bar 与降级链中预演。

## 遗留与后续

- 变更广播的发布端（tokio broadcast）随 M2 摄取接线；TUI 切订阅路径时轮询代码即降为纯降级分支。
- `session.svg` 交互录屏为逐帧离散切换（1.6s/帧）；若维护者想要真终端逐键录制，可在 CI 或人工环境用 asciinema（本机自动化受前台锁限制）。
- 桌面 Tauri 的 IPC 事件承载 ChangeEvent 结构（M5）。
- Windows Terminal 交互截图全套（历史/诊断视图真机版）可在维护者空闲时以 `d` 一键自查，或由后续会话在无人桌面重拍。
