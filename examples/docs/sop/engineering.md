# Engineering SOP

## Change control
- One logical change per PR.
- Run project test and lint commands before requesting review.
- Do not commit secrets, `.env`, or personal paths from `~/.codex/`.

## Verification
- Every Done item in AGENTS.md must map to a command in Commands.
- Prefer automated checks over manual "looks good".

## Security
- No credentials in source, logs, or AGENTS.md.
- Dependencies: prefer maintained packages; note license for commercial use.

## Agent usage
- Read `docs/design/DESIGN.md` before UI changes.
- Read language SOP (`docs/sop/python.md`, etc.) before editing that stack.
