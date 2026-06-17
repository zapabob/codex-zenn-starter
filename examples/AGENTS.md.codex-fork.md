# AGENTS.md

## Project
Upstream-first fork of openai/codex. Official baseline: https://github.com/openai/codex

## Commands
- Build CLI: `cargo build -p codex-cli` (from `codex-rs/`)
- Test core: `cargo test -p codex-core`
- Config schema: `just write-config-schema` (after ConfigToml changes)
- Upstream sync: `python scripts/upstream_sync.py` (maintainers only)

## Rules
- Extension code belongs in `plugins/`, `extensions/`, or `zapabob/` — not inline in upstream-owned core.
- Do not modify `CODEX_SANDBOX_*` test behavior without reading AGENTS.md sandbox notes.
- Do not commit secrets, tokens, or personal paths from `~/.codex/`.
- Match upstream coding style (clippy, argument-comment-lint where applicable).

## Upstream
- Sync driver: `scripts/upstream_sync.py`
- Changelog: `CHANGELOG.md`
- Compare: `openai:main...your-fork:main` on GitHub

## Done
- Tests pass for affected crates.
- No unintended diffs in upstream-owned directories.
- CHANGELOG updated if release-visible behavior changed.
