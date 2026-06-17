# AGENTS.md

## Project
Rust CLI / library workspace. Main crates under `codex-rs/` or project-specific `crates/`.

## Commands
- Build: `cargo build`
- Test: `cargo test`
- Lint: `cargo clippy -- -D warnings`
- Format: `cargo fmt --all`

## Rules
- Crate names use project prefix if monorepo (e.g. `myapp-core`).
- Do not modify `Cargo.lock` unless adding/updating dependencies intentionally.
- Do not change sandbox-related test env vars without understanding CI implications.
- Run `cargo fmt` before finishing.

## Done
- `cargo test` passes for affected crates.
- `cargo clippy -- -D warnings` passes on affected crates.
- Response includes crate names touched and test commands run.
