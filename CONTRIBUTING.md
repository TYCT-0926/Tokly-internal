# 参与贡献

先读三份文档：[开发流程](docs/开发流程.md) · [设计原则](docs/设计原则.md) · [ADR 目录](docs/adr/)

## 快速规则

- Spec 先行：adapter 先看 `docs/sources/<name>.md`，格式文档不全就先补文档
- Conventional Commits，`main` 永远可发布，一个 PR 一件事
- CI 全绿（lint / typecheck / test / 三平台 build）才可合入
- TypeScript `strict`，不留 `any`、死代码、TODO 占位
- 注释只写非显而易见的"为什么"，英文；文档用中文
- 采集相关 PR 必须在描述中声明不违反 [ADR-0003（只读零侵入）](docs/adr/0003-只读采集原则.md)
- 文档同步是门禁不是提醒：行为变更的 PR 必须同 PR 更新相关文档（见 [文档规范](docs/文档规范.md)）
- UI 变更对照 `docs/设计原则.md` 与设计验收清单自查：禁止清单里的东西一样都不许出现
- 界面文案改动需同步 zh-CN 与 en 两份语言文件（见 [ADR-0007](docs/adr/0007-国际化.md)）
