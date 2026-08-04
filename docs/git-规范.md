# Git 规范

目标：历史读起来像一位严谨的工程师在讲故事——每个 commit 都有存在的理由。

## 入库与不入库

**入库**：源码、测试、fixtures（仅人工构造的脱敏样本）、锁文件（`Cargo.lock` 与 `pnpm-lock.yaml` 必须入，保证可复现构建）、CI 配置、文档公共子集。

**不入库**（.gitignore 已覆盖，违规即 CI 红）：`node_modules/`、`dist/`、`.turbo/`、`coverage/`、任何真实用户数据样本、密钥与凭据（连测试用的假凭据也走环境变量）、本地覆盖配置（`*.local.*`）、OS 垃圾（`.DS_Store`、`Thumbs.db`）。

**永不入库**：对话内容样本、用户主目录路径的真实快照、任何 token/cookie。fixtures 一律人工构造，禁止从真实数据导出后"忘记脱敏"。

`internal/` 与 `docs/` 永不入公开仓：CI guard job 对 `git ls-files` 把关，树内出现即红。

## Commit

- Conventional Commits：`feat(core): ...` / `fix(adapter-claude): ...` / `docs(adr): ...` / `refactor:` / `test:` / `chore:`，scope 用包名
- 标题 ≤72 字符，祈使句，英文（代码仓库的国际惯例）；正文写动机与取舍，不写改动复述
- 一个 commit 一个逻辑单元；`wip`/`fixup` 不许进 main（squash 或 rebase 清理）
- **禁止任何工具署名**：不加 `Generated with …`、`Co-Authored-By: <bot>`、AI 工具标记。commit 的作者与语气就是工程师本人；guard job 在 CI 对 `origin/main..HEAD` 的提交正文机器拦截
- 不提交注释掉的代码、调试输出、`console.log` 残骸

## 分支与 PR

- `main` 永远可发布、永远绿；功能分支 `feat/<scope>`、修复 `fix/<scope>`，生命周期 < 3 天
- PR 标题即最终 squash commit 标题；描述按模板：解决什么、为什么这样解、测试证据
- 一个 PR 一件事：<400 行 diff 为宜，超出先拆
- review 讨论解决后才能合入；不 self-merge（个人阶段例外，但 checklist 照走）

## 标签与发布

- 语义化版本；tag 用 annotated（`git tag -a v0.2.0 -m "…"`），不用 lightweight
- tag 只在 CI 绿的 main 上打；发布 = release PR 统一更新版本与 CHANGELOG → tag → cargo-dist / Tauri / npm 发布产物
- CHANGELOG 在 release PR 中维护：去掉内部噪音，确保用户能读懂每一条

## 历史卫生

- main 的历史是产品的一部分：读 `git log` 应该能重构出每个决策的"为什么"
- 合入前作者自己先 `git rebase -i` 清理碎 commit（"oops"、"fix typo" 不该存在于 main）
- force-push 只允许在自己的功能分支，main 永不
- 禁用 `git clean -xdff`（会连 ignored 目录一起删）；清理构建产物用 `cargo clean`，确需清未跟踪文件用 `git clean -xdf -e internal`
