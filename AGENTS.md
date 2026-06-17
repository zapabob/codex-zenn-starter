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

## CI
- PR review: `.github/workflows/codex-pr-review.yml` (requires `OPENAI_API_KEY` secret)
- Config: `.github/codex/home/config.toml`
- Prompt: `.github/codex/prompts/review.md`

## Done
- `npm run preview` shows expected pages at http://localhost:8000
- Frontmatter valid for Zenn (title, emoji, topics, published)
- git push triggers Zenn deploy when GitHub integration is enabled
