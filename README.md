# Tokly

Local-first, open-source AI token usage analytics.

One local dataset, three ways to open it: **Web Dashboard** (the main interface) · **TUI** (live terminal monitoring) · **Desktop** (system tray + pinnable widget + full window).

Covers token consumption, cost, sessions, and tool calls across AI coding tools: Claude Code, Codex CLI, OpenCode, Gemini CLI, Copilot, Cursor.

## Product principles

1. **Read-only, zero intrusion** — Tokly never modifies any tool's config, proxy, or environment variables. It only parses local data read-only; official APIs require explicit user authorization.
2. **Local warehouse, immune to history loss** — full import on first run, incremental ingestion afterwards, retained forever. Source tools cleaning their logs (Claude deletes after 30 days by default) does not affect Tokly's data.
3. **Provenance labels** — every number is labeled by origin: `native` (cost reported by the source tool) / `computed` (calculated from pricing tables) / `estimated` (approximation). No fake precision.
4. **One core, three frontends** — a normalized data model + local SQLite as the single source of truth; Web/TUI/Desktop are all views onto it.
5. **Performance by architecture, not rescanning** — watermark-based incremental ingestion; no ccusage-style full rescans on every run.
6. **Design is an engineering goal** — one set of design tokens feeds all three renderers: restrained, precise, no AI-flavored visuals (see `docs/设计原则.md`).

## Repository structure

> **Status: docs-only.** The code is being rewritten according to these documents — `apps/` and `packages/` below describe the target layout and do not exist yet.

```
apps/        # cli · web (Vite SPA) · tui (OpenTUI) · desktop (Tauri 2)
packages/    # core (ingest/normalize/store) · pricing · adapter-* · design-tokens
docs/        # architecture · design specs · dev process · ADRs · source specs · research
```

## Documentation (mostly Chinese)

- Architecture overview: [docs/架构总览.md](docs/架构总览.md)
- Roadmap & priorities: [docs/路线图.md](docs/路线图.md)
- Development process: [docs/开发流程.md](docs/开发流程.md)
- Design principles: [docs/设计原则.md](docs/设计原则.md) · Design specs: [docs/design/](docs/design/)
- Architecture decision records: [docs/adr/](docs/adr/)
- Database schema (normative DDL): [docs/数据库-schema.md](docs/数据库-schema.md)
- Data source format specs: [docs/sources/](docs/sources/)
- Competitive research: [docs/research/竞品调研.md](docs/research/竞品调研.md)

## Privacy

Tokly uploads nothing and never stores conversation content (token/cost metrics only). Everything stays on your machine. See [SECURITY.md](SECURITY.md).

## License

[MIT](LICENSE)

---

## 中文简介

Tokly 是本地优先的开源 AI token 统计工具：一份本地数据，三种打开方式（Web Dashboard / TUI / 桌面端），覆盖 Claude Code、Codex CLI、OpenCode、Gemini CLI、Copilot、Cursor 的 token 消耗、成本、会话与工具调用。只读零侵入、永久历史仓库、逐数字溯源标注。仓库当前为纯文档状态（代码按文档重写中），文档以中文为主。MIT 许可。
