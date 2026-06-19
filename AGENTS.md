# AGENTS.md

## Project
Zenn book starter for OpenAI Codex practice guide. Markdown content under `articles/` and `books/`.

## Commands
- Install: `npm ci`
- Preview: `npm run preview`
- New article: `npm run new:article`
- New book: `npm run new:book`

## Rules
- Reader-facing tone in `articles/` and `books/` — no author-internal notes.
- Do not set `published: true` unless release is intended.
- Templates live in `examples/`; copy, do not symlink into published chapters.
- Do not commit secrets, API keys, or personal paths from `~/.codex/`.
- Implementation logs go to `_docs/`; not part of Zenn published content.

## CI PR review (Codex)
When reviewing a PR via `codex exec review --base`, respond in Markdown:

1. **Summary** — 1–2 sentences on what changed
2. **Review** — risks, test gaps, or suggestions (bullets OK)

Follow Commands/Rules above. Do not suggest unrelated refactors. If tests were not run, say what to verify.

## CI setup
- PR review: `.github/workflows/codex-pr-review.yml` (ChatGPT auth via `CODEX_AUTH_JSON` secret)
- Config: `.github/codex/home/config.toml`
- Prompt: `.github/codex/prompts/review.md`
- Local auth: `~/.codex/auth.json` — never commit; use `gh secret set CODEX_AUTH_JSON` for CI

## Done
- `npm run preview` shows expected pages at http://localhost:8000
- Frontmatter valid for Zenn (title, emoji, topics, published)
- git push triggers Zenn deploy when GitHub integration is enabled

## Learned User Preferences
- Keep `articles/` and `books/` free of author-to-author advice, planning notes, internal nicknames, and publishing strategy (price/page targets); use `_docs/` or non-published drafts for internal notes.
- When expanding Codex chapters, deep-research official OpenAI Codex docs and zapabob/codex before drafting.
- Validate `examples/` CI/workflows in this repository before writing abstract CI or fork chapters.
- Prefer ChatGPT/Codex sign-in auth (`~/.codex/auth.json` → `CODEX_AUTH_JSON` in CI) over exposing `OPENAI_API_KEY` in workflows.

## Learned Workspace Facts
- Zenn deploys from GitHub `zapabob/codex-zenn-starter` on branch `main` to zenn.dev/zapabob.
- Primary book slug is `openai-codex-design-book`; teaser article slug is `codex-book-teaser`.
- Book scope centers on AGENTS.md, Sandbox, Approval, CI, and zapabob/codex as an upstream-first fork case study.
- `examples/` holds copyable AGENTS.md templates and `.github/` workflow patterns for readers.
