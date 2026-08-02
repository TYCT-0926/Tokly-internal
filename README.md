# Tokly 内部仓库（tokly-internal）

Tokly 的战略与设计文档仓，**私有，不放代码**。代码与公共文档子集在公开仓 [Tokly](https://github.com/TYCT-0926/Tokly)（本地 `E:/Axiom/Tokly-public`）。

Tokly：本地优先、开源（MIT）的 AI token 使用统计与可视化——六源 → 归一化 → SQLite 永久仓库 → Web/TUI/桌面三端。

## 内容地图

| 路径 | 内容 |
|---|---|
| `docs/adr/` | 架构决策记录（战略取舍，含竞品判断；只追加不修改） |
| `docs/research/` | 竞品调研与外部审查报告（档案，冻结打戳） |
| `docs/design/` | 设计图纸：令牌系统 / 主题 / 组件规格（照此施工，不得即兴） |
| `docs/架构总览.md` | 一页看懂系统：形态、数据流、包职责、进程模型、架构约束 |
| `docs/数据库-schema.md` | 规范 DDL（M0.5 冻结门交付物） |
| `docs/路线图.md` | 里程碑、优先级、性能 SLO |
| `docs/风险登记.md` | 风险台账与对冲措施 |
| `docs/仓库策略.md` | 开源/闭源边界与双仓同步规则（本内容地图的依据） |
| `docs/文档规范.md` | 文档元规范：SSOT、分寿命、时效打戳（公共子集，公开仓同步） |

数据源格式 spec（`docs/sources/`）、设计原则、开发流程、git 规范、代码风格已迁至公开仓 `docs/`，本仓不复制；引用一律写"公开仓 docs/xxx"。

## 边界规则（一句话）

内部子集（`docs/adr/`、`docs/research/`、`docs/design/`、路线图、风险登记、架构总览、仓库策略）**永不进入公开仓**，包括 git 历史。

## 协作者准则

人类与 AI 协作者的行为准则见 [AGENTS.md](AGENTS.md)（公开贡献者版在公开仓）。
