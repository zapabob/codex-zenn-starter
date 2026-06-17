# AGENTS.md

## Project
Application repository with Codex MCP servers configured in `.codex/config.toml`.

## Commands
- Install: see README
- Test: `npm test` or project-specific test command
- Lint: project lint command

## MCP usage
- Prefer repo-scoped MCP defined in `.codex/config.toml` over ad-hoc curl.
- Before calling GitHub MCP tools, confirm issue/PR numbers from user input.
- Do not exfiltrate secrets via MCP tool arguments or logs.
- If an MCP server fails to start, report the server name; do not bypass with `danger-full-access`.

## Rules
- Do not add `[mcp_servers.*]` entries that embed API keys in `config.toml`; use `env` or `bearer_token_env_var`.
- Use `enabled_tools` / `disabled_tools` to limit scope per server.
- Set `default_tools_approval_mode = "prompt"` for servers that mutate external state.
- Do not read `.env` through filesystem MCP roots.

## Done
- Tests pass.
- MCP tool usage is mentioned in the final summary when external systems were queried.
- No secrets printed in responses.
