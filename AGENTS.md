# Repository Guidelines

## Project Structure & Module Organization

This repository is a Markdown learning wiki for AI agent engineering. It does not contain the source code of the referenced projects.

- `raw/`: immutable source learning paths. Treat these files as read-only inputs.
- `wiki/`: generated and maintained notes. `wiki/index.md` is the authoritative table of contents and status tracker; `wiki/log.md` is append-only.
- `memory/`: durable learning preferences and process notes for future sessions.
- `README.md`: public overview and usage guide.
- `CLAUDE.md`: existing agent workflow and note conventions.

Use kebab-case for new wiki pages, for example `wiki/tool-calling.md` or `wiki/context-management.md`.

## Build, Test, and Development Commands

There is no build system or application runtime. Common maintenance commands are:

- `rg "term" wiki raw`: search notes and source material.
- `find wiki -maxdepth 1 -type f | sort`: list wiki pages.
- `git diff -- README.md wiki/index.md`: review pending documentation edits.

Before finishing changes, verify Markdown links and confirm `wiki/index.md` matches the actual pages in `wiki/`.

## Coding Style & Naming Conventions

Write concise Markdown with clear headings, short paragraphs, and project-specific examples. New note pages should focus on one capability or concept. Follow the existing status flow: `⬜ 待开始` → `🟡 进行中` → `🟢 完成`.

For comparison notes, use the established blockquote style:

```markdown
> 💡 对比：...
```

Do not rewrite `raw/` files during normal note updates.

## Testing Guidelines

No automated tests are configured. Treat documentation review as the test process:

- Check internal links between wiki pages.
- Ensure referenced source paths are clearly identified as belonging to `cc-haha` or `OpenClaw`, not this repository.
- Keep `wiki/log.md` append-only when recording work.

## Commit & Pull Request Guidelines

Recent history uses Conventional Commits, especially `docs:` and `feat:` prefixes, such as `docs: add README with repo overview and usage guide`. Keep commit subjects imperative and scoped to the change.

Pull requests should include a short summary, changed pages, related learning topic, and any status updates made in `wiki/index.md`. Include screenshots only when visual rendered Markdown differences matter.

## Agent-Specific Instructions

For learning sessions, follow the workflow in `CLAUDE.md`: read relevant `raw/` sections, update or create `wiki/<topic>.md`, refresh `wiki/index.md`, then append an entry to `wiki/log.md`.
