# 実装ログ: 章6 MCP / 章7 CI / examples 同梱

- **日時**: 2026-06-17
- **ブランチ**: main

## 実施内容

| 成果物 | 内容 |
|--------|------|
| `mcp-integration.md` | `[mcp_servers.*]`、codex mcp CLI、agents YAML、AGENTS.md 連携 |
| `ci-codex-action.md` | openai/codex-action、codex-home、AGENTS.md 分離 |
| `examples/` | 5種 AGENTS.md テンプレ + global-vs-project + CI サンプル |
| ルート `AGENTS.md` | codex-zenn-starter 自身の Codex ルール |

## ~/.codex/AGENTS.md との整理

- ユーザー `C:\Users\downl\.codex\AGENTS.md` は **グローバル設定**（1600行超、文体・スキル・MCP方針）
- リポジトリには **実行可能な短い AGENTS.md テンプレ** のみ同梱
- `examples/global-vs-project.md` に使い分けを文書化
- 個人 config の API キーは examples に **含めていない**

## 参照ソース

- https://developers.openai.com/codex/mcp
- https://developers.openai.com/codex/github-action
- https://github.com/zapabob/codex/.codex/agents/code-reviewer.yaml
- openai/codex `.github/codex/labels/codex-review.md`

## config.yaml

章6 `mcp-integration`、章7 `ci-codex-action` を追加（全7章）
