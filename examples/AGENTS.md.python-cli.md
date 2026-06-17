# AGENTS.md

## Project
Python CLI application managed with uv.

## Commands
- Install: `uv sync`
- Test: `uv run pytest`
- Lint: `uv run ruff check .`
- Format: `uv run ruff format .`
- Typecheck (if configured): `uv run pyright`

## Rules
- Do not read or print secrets from `.env` or `.env.local`.
- Do not modify files under `data/raw/` or `tests/fixtures/golden/`.
- Keep changes small; one logical change per commit message.
- Prefer editing `src/` over adding scripts at repo root.

## Done
- `uv run pytest` passes.
- `uv run ruff check .` passes.
- Final response lists changed files and verification commands run.
