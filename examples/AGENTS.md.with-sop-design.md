# AGENTS.md

## Project
Example repo using SOP + DESIGN.md split. Copy `examples/docs/` into your project.

## Commands
- Install: `npm ci`
- Test: `npm test`
- Lint: `npm run lint`

## Rules
- Follow `docs/sop/engineering.md` for all changes.
- UI work: read `docs/design/DESIGN.md` before editing components.
- Language-specific: `docs/sop/python.md`, `docs/sop/typescript.md` when present.
- Do not commit secrets or symlink targets outside the repo.

## Symlink option (optional)
- Windows: `New-Item -ItemType SymbolicLink -Path AGENTS.md -Target docs\sop\AGENTS.full.md`
- Unix: `ln -sf docs/sop/AGENTS.full.md AGENTS.md`
- Prefer explicit Rules paths if symlinks are awkward on your team.

## Done
- Tests and lint pass per Commands.
- UI changes align with DESIGN.md.
