# 阶段七A · Web UX 竖切（spike ②）执行报告

- 日期：2026-08-04
- 分支：`feat/web-slice`（worktree `../Tokly-web-slice`），基线 main `b660db6`
- 范围：apps/web 全新 + `.github/workflows/ci.yml` web/web-e2e 两 job + pnpm-lock
- 交付物：本报告 · `assets/七A-C/` 截图 10 张 + 首载编排录屏 + **`assets/七A-C/offline.html`（双击即看，mock 内联）**

## 一、完成定义输出摘要（全绿）

| 命令 | 结果 |
|---|---|
| `pnpm typecheck` | 通过（tsc -b，app+node 双项目，strict 全开） |
| `pnpm lint` | 通过（eslint 0 问题含 React Compiler 规则；stylelint 五条自定义规则 0 违例） |
| `pnpm test` | 37/37（生成器确定性/右偏/分位分档/MockLive 版本语义/门禁自测 14 条） |
| `pnpm build` | 通过（tsc → vite → 首屏图谱检查 → size-limit → 离线单文件 → 打包） |
| `pnpm playwright` | 25 用例：24 直过 + 1 条 @perf 编排时序在负载下经一次重试过（容噪机制，见 §六） |

axe：amber-dark / amber-light / mono-dark / mono-light 四变体概览 + amber 双模式下钻页，**零 violation**。

## 二、任务要求的六项实测结论

### 1. SVG 渲染器 dataZoom 实测 → **SVG 定格，无需回退 canvas**

| 数据量 | 帧数 | 平均帧 | p95 | 最差帧 | 连续超帧 |
|---|---|---|---|---|---|
| 91 点 × 3 系列（90d 日桶） | 175 | 16.67ms | 16.8 | 16.8 | 0 |
| **1096 点 × 3 系列（3 年全窗）** | 177 | **16.67ms** | 16.7 | 16.8 | **0** |

滚轮缩放进出各 15 档全程满帧。SVG 渲染器在本产品数据量级（日桶为主、≤3 系列）没有任何余量压力。

### 2. 虚拟化判决 → **分层：分组层 React Aria，事件层回退 TanStack Virtual**

React Aria Table + Virtualizer 在 10 万行下的实测（判决依据）：

| 指标 | React Aria Virtualizer | TanStack Virtual（回退后） |
|---|---|---|
| 导航→可交互 | **8209ms** | **739ms**（含 13.6MB fetch ~300ms） |
| 滚动平均帧 | 23.56ms（~42fps） | **16.67ms（满帧）** |
| 滚动最差帧 | 83.3ms | 16.8ms |
| 连续超帧 | 2（贴线） | 0 |
| DOM 行数 | ~21 | ~28–41 |

根因：RAC 的 collection 构建是**全量物化**的——Virtualizer 只省 DOM 不省 collection，10 万行的节点构建约 7.5s 落在主线程。判决：**事件层（原始日志，行数无上界）用 TanStack Virtual**（`EventLogTable`，手写 grid 语义：aria-rowcount/rowindex、columnheader、sticky 表头、translateY 定位）；**分组层（模型/项目/会话，行数有界）保留 RAC Table**——它的键盘导航、焦点管理、读屏语义值回票价。回退面收敛在一个组件，DTO 不变。

### 3. React Compiler × React Aria 兼容问题清单 → **零问题（显式声明）**

- babel-plugin-react-compiler 1.0.0 × react-aria-components 1.20.0 × React 19.2.8；
- eslint `react-hooks` recommended-latest（含 compiler 规则）0 告警——无 bailout、无违例；
- 25 条 e2e（含 RAC Table/RadioGroup/DateRangePicker/Popover 交互路径）全过；
- 唯一注意项（非 Compiler 问题）：RAC Radio 的可见交互面是包裹 label，测试点击须瞄 label 文本而非隐藏 input（已封装 `pickSegment` helper）。

### 4. 首屏体积

| 项 | 数字 |
|---|---|
| 入口 chunk（唯一首屏 js） | 780KB raw / **245KB gzip / 208KB brotli**（size-limit 预算 256KB） |
| 首屏 css | 33KB raw / 6.2KB gzip（含全部 10 变体令牌） |
| echarts chunk（懒加载，永不进首屏） | 526KB raw / 181KB gzip |
| 首屏图谱检查 | 1 chunk，无 echarts/zrender（manifest 静态导入图遍历 + 源码扫描，CI 强制） |
| 离线单文件 | 1.59MB（js+css+字体 全内联） |

入口偏重的大头是 react-aria-components + 双 TanStack；对 spike 可接受，7B 可做路由级再分包（RangeControl/DateRangePicker 仅概览/下钻用）。

### 5. mock 生成器（seed 与分布）

- **seed 20260804**，mulberry32；固定时钟 **2026-08-04T12:00:00Z**；全 UTC。确定性由单测锁定（双跑 digest 相等 + 事件数恰 100,000）。
- 时间包络：1096 天（2023-08-05 起）；日权重 = 增长斜坡 `0.25+0.75(d/D)^1.6` × 星期因子（工作日 0.95–1.15，周末 0.35–0.45）× 噪声(0.7–1.3) + 18 个爆发日(×2.5–4)；日内小时双峰（10–12 / 15–18）+ 晚间尾；最大余数法精确分配 10 万事件。
- 右偏：input ~ LogN(μ7.6,σ1.1)，output ~ LogN(6.3,1.0)，cache 35% 为零否则 LogN(8.5,1.3)；实测 mean/median > 1.5（单测断言）。
- 世界构建的两处刻意设计：**copilot 无项目维度**（落实"0 与无此维度可区分"，表内呈现 `—` + 悬停说明）；**cursor 末 12 天静默**（状态点 ◐ 陈旧态由数据产生，不是 UI 摆拍）。
- 会话：按（源×项目×日）分槽，容量 8–45 事件；成本按每模型 [input,output] 单价（cache 按 input 10%）。

### 6. 首载编排逐帧实测（编排规格 §二）

| 时点（规格） | 实测起变 | 实测落定 |
|---|---|---|
| 顶栏 0ms | 0 | 117 |
| 第一带 80ms | 100（含检测阈 ~1 帧） | 233 |
| 图表容器 240ms | 233 | 400 |
| 热力图签名 400ms | **400（精确）** | 650 |
| 总长 ≤900ms | — | **650ms 全部落定** |

Playwright 断言：偏移 ±1–2 帧、总长 ≤900+1 帧、点击即中断（两帧内全元素 opacity=1 且 data-intro=done）、每会话仅一次、reduced-motion 下逐项动画为 none 且 #root 单次 150ms 淡入。
过程中抓到并修复两个真缺陷：① KPI 骨架→数据的 key 变化导致 article 重挂载、入场动画重播（改稳定槽位 key）；② echarts 懒 chunk 的解析恰落在编排窗口内饿死 rAF——**图表重资源改为编排结束后再挂载**（240ms 淡入的是容器+骨架，编排窗口的帧预算保持神圣）。

## 三、主题与图表同帧（M0.5 定格项）

- **渲染器定格：SVG**（dataZoom 实测满帧，见 §二.1）。
- **var() 直通探针**（Chromium 151）：SVG 表现属性 `fill="var(--x)"` 与 style 属性**均能解析**。但实测揭示决定性约束：**zrender 在 setOption 时替换 SVG 节点**，CSS 过渡（fill/stroke transition）在新节点上无旧值可插值——"图表随 CSS 变量渐变"这条路在 echarts 下不成立，与 var() 是否解析无关。
- **落地机制**：主题切换 = `data-theme` 翻转 + 临时 `data-theme-switching` 属性开启全局 150ms 色彩过渡；图表在**同一 React commit** 内重解析令牌色 setOption 并 **`zr.flush()` 同步落帧**。图表瞬时落新色（窗口首帧），页面 150ms 交叉淡化包裹其外。
- **验收**：Playwright 逐帧断言——不存在"页面已落定新主题而图表仍旧主题"的帧；感知变化窗口（ΔRGB>16 为在变，排除缓动尾）≤150ms+3 帧。跨引擎（FF/Safari）未在本 spike 验证，记为 7B 携带项。

## 四、新鲜度落地（ADR-0019）

### 冲突上报（按红线规则处置，待维护者追认）

ADR-0019 §2「service 提供面向屏的复合查询，**禁止客户端拼多个原子查询**」与本任务原文「Query key 按概览/趋势/热力图/表格分块」**直接冲突**——我先按任务原文实现了四个原子端点，收到 ADR-0019 后依「转述与 ADR 冲突时以 ADR 为准」重构为屏级复合。**已执行侧：ADR**。请追认或裁回。
边界判断（同请裁决）：顶栏数据源状态点属 chrome 而非"一屏数据块"，且与屏内数字无一致性耦合，单列为 `/api/health`（scope=health 失效）；若维护者认为它也应并入屏复合，改动是机械的。

### 版本传播链路

```
MockLive.commit()（写代次 +1，确定性批次）
  → SSE change {data_version, scopes:[events,sessions,rollups]}
  → 客户端 lastSeen 对账：
      v == last+1 → 按 scopes 失效（events/sessions/rollups → ['screen']；health → ['health']）
      v >  last+1 → 全量失效（不补差；本地重取毫秒级）
  → TanStack Query 重取（旧值在屏，无骨架无闪烁）
  → 复合响应单 data_version 落屏；版本递增触发带序级联（KPI→图表→表格，60ms 步进 150ms 淡化）
```

- SSE 重连首事件带当前版本（服务端实现）；心跳 3s，客户端静默阈值 10s（测试可经 sessionStorage 覆写）。
- 编排规格 §四"同一 data_version 内编排"由复合查询结构性保证——一次响应喂全部带。
- mock 控制端点（spike-only）：`POST /api/__mock/commit[?silent=1]`、`POST /api/__mock/heartbeat?on=0|1`，使漏事件与陈旧路径可确定性验证。

### 漏事件恢复的验证方式（Playwright，CI 强制）

两次 `silent=1` 提交（版本前进但事件不发）+ 一次正常提交 → 客户端收到 v4 而 lastSeen=1 → 断言全量失效后 KPI 读数与**全新客户端冷取**逐字符一致（证明追平的是全部三代，不只是被通知的一代）。

### 三态展示（截图在 assets/七A-C/）

| 态 | 触发 | 表现 | 截图 |
|---|---|---|---|
| 新鲜 | 正常事件流 | 安静，无指示 | 新鲜度-新鲜.png |
| 重取中 | 已知新版本、数据在途 | 旧值保持 + 顶栏 6px 呼吸点（禁骨架禁闪烁，Playwright 断言重取期零 skeleton 节点） | 新鲜度-重取中.png |
| 陈旧 | 心跳静默超阈 | 顶栏 `◐ 数据可能过期 · 最后更新 hh:mm`，恢复即清除 | 新鲜度-陈旧.png |
| 快照（离线特例） | file:// 构建无事件源 | 常驻 `快照 · 2026/08/04 12:00`——它本来就是快照，不伪装实时 | 离线快照态.png |

## 五、门禁落地（任务的一半）

### Stylelint 自定义插件（五规则，CI 强制，14 条 fixture 自测）

| 规则 | 内容 |
|---|---|
| tokly/motion-discipline | transition 仅 transform/opacity；禁 `all`；@keyframes 内仅 transform/opacity；规格自身列举的状态反馈色淡入（行 hover/导航/主题交叉淡化）须 `tokly-motion-allow` 注释显式豁免（当前 8 处，每处可数） |
| tokly/duration-tokens | transition/animation 的时长与延迟仅允许 `var(--tokly-dur-*)`/`var(--tokly-spring-ui-duration)`/0；自定义属性带时间字面量同禁；唯一豁免文件 app-tokens.css |
| tokly/no-banned-effects | linear-gradient / backdrop-filter（白名单机制 `tokly-effect-allow` 预留，当前零使用）；radius>12px；shadow blur>24px |
| tokly/spacing-tokens | margin/padding/gap 仅令牌变量/0/auto/calc-over-tokens（区带节奏白名单的值级强制；48/24/12-16 的结构层级靠评审） |
| tokly/no-color-literals | hex/rgb/hsl/oklch 字面量全禁；`color-mix(var 令牌, transparent)` 为唯一放行的 alpha 机制 |

### Playwright（25 用例）与 CI 分层

- **CI（web-e2e job，去 @perf）**：URL 三操作（直链/刷新/后退 + 非法参数塌陷默认）、漏事件全量失效、陈旧标注出现与恢复、主题同帧逐帧断言、四变体端到端、编排中断/仅一次/reduced-motion、axe 五扫描。
- **本地 @perf 门**（墙钟敏感，共享 runner 不可靠，数字入本报告）：三场景帧预算（热力图首播/下钻生长/范围切换，"超帧"= 帧间隔 >25ms 即掉 vsync，连续 >2 帧即败）、100k 滚动、dataZoom、编排逐帧时序表。本地配置 retries:1 容纳开发机噪声。
- size-limit：入口 256KB 预算（实测 208KB brotli）+ 首屏图谱无 echarts 检查，进 build 链。

## 六、设计自查（反模板设计守则 §五）

### 模仿什么真实物件

一台**台式测量仪器**——示波器的屏、万用表的读数区：上缘是量程与通道拨键（时间范围/主题/信号灯），主区是读数、迹线与年度热力签名，下缘是原始采样日志。三处体现：① KPI 无卡片直排画布、发丝竖线分隔，是仪表读数区不是 SaaS 卡片；② 顶栏数据源状态点用 `●○◐!` 单宽字符——与 TUI 状态字符同一套语义词汇，一眼同一台仪器；③ 溯源标注（计算值）与快照/陈旧徽标——仪器永远告诉你读数的口径和新鲜度。

### 八条 AI 味症状逐项

1. 均匀圆角卡片阵：**未命中**——全页唯一有边界的是主角图表（发丝框），KPI/热力图/表格直接排在 canvas 上。
2. 标题+图标+副标题三件套：**未命中**——图标仅状态点（替代文字的语义字符），副标题不存在；带标题只在需要区分处（节奏/花在哪了）。
3. 等分栅格：**命中即改**——KPI 带初版 1fr 等分，按信息权重改为 1.5fr/1.2fr/1fr（tokens > 成本 > 会话）。
4. 色彩做分类：**未命中**——图表第二系列灰、第三系列同相明度变体；类别靠位置与排序。
5. 动效均匀涂抹：**未命中**——表现力预算只在首载编排+热力图首播；其余全部状态反馈（每处豁免注释可 grep 计数：8 处）。
6. 数字当文本排版：**未命中**——全局 tabular-nums，一切读数 Maple Mono 右对齐，成本恒两位小数（小数点成列），紧凑记数只在卡片与轴。
7. 空态插画+鼓励语：**未命中**——空窗一句话+一个动作；超保留窗是说明+调整入口（组件规格 §6 原文文案）。
8. 首屏 hero：**未命中**——产品名 13px 在导航角落，首屏第一带就是读数。

### 十项细节落地

1. 光学对齐：delta 箭头与百分比 baseline 对齐；状态点 12px 单宽字符垂直居中于 16px 行。**部分主观**，见存疑项。
2. 小数点成列：成本列恒两位小数 + tabular-nums + 右对齐 ✓。
3. hover 不位移：全部 hover 仅色彩（lint 强制——位移类属性根本过不了 motion-discipline）✓。
4. 焦点环不裁切：表格行/分段控件用负 offset 内环；滚动容器自身可聚焦承载外环 ✓。
5. CLS=0：骨架与内容等形等高（KPI 槽位固定行高、图表/热力图/表格盒定高）；KPI 稳定 key 修复了数据到达重挂载；状态点/徽标区 min-size 预留 ✓。
6. 发丝线：全部走 `--tokly-border-hairline`（alpha 基），1px 常量 `--tokly-hairline-w` ✓（2x 屏 0.5px 物理线未做，7B 可用 0.5px 媒查升级——记待办）。
7. 长文本策略：会话 id 中段缩略（middleEllipsis，头尾各半）；表格单元溢出末端省略 + title 全文；时间列宽保证永不截断。一致可预测 ✓。
8. 0 与无此维度：copilot 项目列 `—`+悬停"该源不提供项目维度"；热力图未来日**不渲染格子**（缺席≠零活动）；零活动日 heat[0] ✓。
9. 键盘回路：RAC 承担分组表格/分段控件/日期选择（弹层关焦点回触发器）；Tab 序=视觉序；**热力图格子无键盘路径**（等价物承载）——见存疑项。
10. 两主题各自调过：浅色反向强度热力图、hairline alpha 分深浅（令牌层）、激活段 hairline 描边专为浅色 raised=overlay=白的补偿（§7 修订）；axe 四变体分别过 ✓。

### 主观存疑项（数值合规、观感待裁决）

1. **mono-dark 的多系列可分性**：单色相纪律下三条迹线只靠明度区分，交叉密集段偏吃力。备选：mono 下降为单系列+灰参考线。请人工验收 概览-mono-dark.png。
2. **活跃会话 KPI 无 sparkline**：槽位保留（对齐不塌）但视觉上第三列偏空。我的判断：可接受（信息权重最低），但"留白"与"缺料"一线之隔。
3. **图表容器发丝框**：组件规格 §4 未要求边框，守则"主角有边界"允许。我按每屏一主角给了节奏带的图表定界。若维护者觉得"框"重了，去框改留白也成立。
4. **离线快照徽标常驻**：诚实但常驻黄边警示可能读作"坏了"。已用中性措辞（快照·时间戳）区别于陈旧态。
5. **表格行 hover 5% alpha** 在 amber-dark 下极淡（规格值 4–6%）。合规但可感知性差，7B 若调只能改规格值。

### 维护者验收指引

1. **双击 `internal/docs/reports/assets/七A-C/offline.html`**——完整应用，mock 内联（快照态徽标属预期）。看：首载编排（新会话第一次进概览；再看一次需清 sessionStorage 或换浏览器 profile）、四主题切换（图表同帧）、下钻三级（生长/收回方向对称）、事件维度直达 10 万行滚动、自定义日期范围。
2. 截图组按文件名对照 §四/§六；`首载编排.webm` 是 0–900ms 时序录屏。
3. 代码侧七A 门禁一眼处：`apps/web/tools/stylelint-tokly.js`（品味即 lint）、`apps/web/e2e/`（六份断言）、`apps/web/src/styles/app-tokens.css`（全部登记值一处）。

## 七、偏差与携带事项（除 §四冲突外）

| # | 事项 | 处置 |
|---|---|---|
| 1 | **fg.faint 可读文本撞 axe AA**（faint 实测 2.25:1；令牌质量门本就只担保 muted≥4.5）| 可读微文本全部升 muted；**组件规格两处待修订**：heat.label 与 provenance.label 的 fg.faint 指定（faint 保留给纯装饰/禁用）|
| 2 | Tier-3 未登记新值（施工需要，先集中在 app-tokens.css 并注明 pending） | 待登记：spark 线宽 1.5px；图表高 280px；表格高 520/360px；事件表列宽 152/120/128/112px；日历格 32px；指示点 6px；分段内缩 2px；快取三档外时长 grow 200ms/return 120ms/breathe 1.2s/snap 1ms/cascade 60ms（其中 200/120 出自动效-编排规格自身参数，与"三档"表述冲突一并请裁）|
| 3 | 路由用 hash history | 离线单文件（file://）强约束；URL 契约与测试不受影响。产品期若必须 path 路由，桌面/serve 可切，离线产物保 hash |
| 4 | 浮层入场用 RAC data-entering 关键帧（等参 150/100ms scale0.95+fade）而非 @starting-style | RAC 管理挂载生命周期，@starting-style 面向无 JS 挂载管理场景；参数一致 |
| 5 | 热力图 tooltip 仅指针可达 | 键盘/SR 走 role=img 摘要 + 表格等价物；7B 如需格级键盘导航再评 |
| 6 | 时间范围"记住选择"（跨会话）未做 | URL 即状态已覆盖分享/复现；localStorage 记忆挂 7B |
| 7 | @perf 帧预算门禁不进 CI | 墙钟敏感；本地门 + 报告数字。CI 跑其余 19 条 |
| 8 | 主题同帧跨引擎（FF/Safari）未验 | spike 仅 Chromium；机制（同 commit setOption+flush）不依赖引擎特性，验证挂 7B |
| 9 | 首屏字体 font-display:swap | 本地毫秒级加载，交换不可感；未做 preload |
| 10 | **TUI 侧交付物（assets/九/）归属假设**：维护者追加消息的 TUI 段按并行的 feat/tui-slice 执行者理解，本报告不含 | 如归属有误请指回 |

## 八、结构速览

```
apps/web/
  vite.config.ts            # compiler/lightningcss/别名/@tokens 前置自检/mock 中间件
  vite.offline.config.ts    # 单文件离线构建（file:// 模块 CORS 约束所致）
  tools/stylelint-tokly.js  # 五条门禁规则（fixture 自测 tools/*.test.ts）
  scripts/                  # 首屏图谱检查 · 离线打包 · 测量 harness
  e2e/                      # url-state / freshness / theme / intro / frame-timing / axe
  src/
    styles/                 # app-tokens（唯一字面量层）+ global（字体/reset/编排/交叉淡化）
    mock/                   # spike-only：prng/generator/aggregate/live(+版本)/node/server(SSE)
    api/                    # client（http|file:// 双运输）/queries（屏级）/freshness（三态+对账）
    app/                    # router(hash)/search 校验/theme/intro/drillMotion/Shell
    components/             # KpiBand/TrendChart(懒)/Heatmap/DrillTable(RAC)+EventLogTable(TV)/
                            # RangeControl/SegmentedControl/ThemeSwitch/SourceDots/FreshnessBadge/…
    routes/                 # Overview / Drill / Settings(占位)
```
