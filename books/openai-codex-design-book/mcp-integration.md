---
title: "MCPでCodexを外部ツールとつなぐ"
free: true
---

# MCPでCodexを外部ツールとつなぐ

MCP（Model Context Protocol）は、Codex に **外部ツールとコンテキスト** を接続するための標準です。

GitHub の issue 操作、ドキュメント検索、ブラウザ制御、Linear のチケット参照など、Codex 単体ではできない操作を MCP 経由で追加できます。

この章では、

1. `[mcp_servers.*]` による MCP 設定
2. `.codex/agents/` によるサブエージェント定義（zapabob/codex 実例）
3. AGENTS.md との役割分担

を扱います。

## MCP が解決すること

第3章の AGENTS.md は「このリポジトリでどう作業するか」を伝えます。  
第4章の Sandbox は「どこまでファイルとコマンドを許可するか」を決めます。

MCP はその上に、**外部システムへのアクセス経路** を足します。

| レイヤー | 設定場所 | 役割 |
|---------|---------|------|
| リポジトリルール | `AGENTS.md` | test / lint / 禁止事項 |
| 権限境界 | `config.toml` の sandbox / approval | ローカルファイル・シェル |
| 外部ツール | `[mcp_servers.*]` | GitHub API、ドキュメント MCP 等 |
| サブエージェント | `.codex/agents/*.yaml` | 専門タスクの並列実行（フォーク拡張） |

## 設定ファイルの置き場所

MCP 設定は `config.toml` の `[mcp_servers.<name>]` テーブルに書きます。

| ファイル | スコープ |
|---------|---------|
| `~/.codex/config.toml` | 全プロジェクト共通 |
| `your-repo/.codex/config.toml` | そのリポジトリのみ（trusted project） |

CLI と IDE 拡張は **同じ config.toml を共有** します。一度設定すれば、TUI と IDE の両方で MCP が使えます。

## CLI で MCP サーバーを追加する

```powershell
codex mcp add context7 -- npx -y @upstash/context7-mcp
```

環境変数付き:

```powershell
codex mcp add myserver --env TOKEN=placeholder -- npx -y @scope/my-mcp-server
```

TUI 内では `/mcp` でアクティブなサーバー一覧を確認できます。

```powershell
codex mcp --help
```

## config.toml で STDIO サーバーを定義する

STDIO サーバーは、Codex が子プロセスとして起動するローカル MCP です。

```toml
# ~/.codex/config.toml または .codex/config.toml

[mcp_servers.context7]
command = "npx"
args = ["-y", "@upstash/context7-mcp"]

[mcp_servers.github]
command = "npx"
args = ["-y", "@modelcontextprotocol/server-github@latest"]
env_vars = ["GITHUB_TOKEN"]
```

:::message alert
API キーを `config.toml` に直書きしないでください。`env_vars` で環境変数名だけ指定し、値は OS 環境または CI の secrets から渡します。
:::

### ファイルシステム MCP（読み取りルート限定）

```toml
[mcp_servers.filesystem]
command = "npx"
args = [
  "-y",
  "@modelcontextprotocol/server-filesystem@latest",
  "C:\\Users\\you\\Projects",
]
```

アクセス可能なパスを **明示的に列挙** します。ホームディレクトリ全体を渡すのは避けてください。

## HTTP / Streamable HTTP サーバー

リモート MCP は `url` で指定します。

```toml
[mcp_servers.linear]
url = "https://mcp.linear.app/mcp"

[mcp_servers.openai_docs]
url = "https://developers.openai.com/mcp"
```

Bearer トークン:

```toml
[mcp_servers.example]
url = "https://api.example.com/mcp"
bearer_token_env_var = "EXAMPLE_MCP_TOKEN"
```

OAuth が必要なサーバー:

```powershell
codex mcp login <server-name>
```

## ツール単位の承認

MCP ツールも Sandbox と同様、**承認ポリシー** を設定できます。

```toml
[mcp_servers.github]
command = "npx"
args = ["-y", "@modelcontextprotocol/server-github@latest"]
env_vars = ["GITHUB_TOKEN"]
default_tools_approval_mode = "prompt"

[mcp_servers.github.tools.create_issue]
approval_mode = "approve"
```

| 値 | 意味 |
|----|------|
| `auto` | 確認なしで実行 |
| `prompt` | 実行前に確認 |
| `approve` | 明示承認が必要 |

外部状態を変更するツール（issue 作成、デプロイ、ファイル書き込み）は `prompt` 以上を推奨します。

### ツールの allow / deny

```toml
[mcp_servers.playwright]
url = "http://localhost:8931/mcp"
enabled_tools = ["navigate", "snapshot"]
disabled_tools = ["screenshot"]
startup_timeout_sec = 20
tool_timeout_sec = 45
required = false
```

`required = true` にすると、そのサーバーが起動できない場合 **Codex 全体の起動が失敗** します。CI では必要なサーバーだけ `true` にします。

## プラグイン bundled MCP

Codex プラグインは MCP サーバーを同梱できます。ユーザー config では on/off とツールポリシーだけ制御します。

```toml
[plugins."github@openai-curated"]
enabled = true

[plugins."github@openai-curated".mcp_servers.github]
enabled = true
default_tools_approval_mode = "prompt"
```

transport（起動コマンド）はプラグイン側が提供するため、ユーザーは **ポリシーだけ** 書きます。

## .codex/agents/ — サブエージェント（zapabob/codex 実例）

[zapabob/codex](https://github.com/zapabob/codex) では `.codex/agents/` に YAML 形式のエージェント定義があります。

例: `code-reviewer.yaml`

```yaml
name: "code-reviewer"
goal: "Analyze code for type safety, security, and best practices."
tools:
  mcp:
    - grep
    - read_file
    - codebase_search
  fs:
    read: true
    write:
      - "./artifacts"
  net:
    allow:
      - "https://docs.rs/*"
  shell:
    exec:
      - cargo
      - npm
policies:
  secrets:
    redact: true
artifacts:
  - "artifacts/code-review-report.md"
```

これは **MCP サーバー定義とは別レイヤー** です。

| 概念 | 役割 |
|------|------|
| `[mcp_servers.*]` | どの MCP サーバーを起動するか |
| `.codex/agents/*.yaml` | 特定タスク用サブエージェントの goal / tools / 成果物 |

公式 openai/codex でも subagents 機能は進化中です。フォークの YAML は **拡張の置き方の参考** として読んでください。

## AGENTS.md に書く MCP ルール

MCP 設定だけでは足りません。AGENTS.md に **使い方の規律** を書きます。

`examples/AGENTS.md.with-mcp.md` を参照してください。要点:

- 秘密情報を MCP 引数に載せない
- 外部変更系ツールは承認モードを `prompt` 以上に
- MCP 失敗時に `danger-full-access` で迂回しない
- 外部 API を触ったら最終報告に明記する

## ~/.codex/AGENTS.md との関係

個人環境の `~/.codex/AGENTS.md` には、スキル一覧や MCP 利用方針が書かれることがあります。これは **ユーザー全体の振る舞い設定** であり、リポジトリにコミットしません。

| ファイル | 用途 |
|---------|------|
| `~/.codex/AGENTS.md` | 個人の文体・スキル・MCP 方針 |
| `repo/AGENTS.md` | チーム共有の test / lint / 禁止事項 |

詳細は `examples/global-vs-project.md` を参照してください。

## 最小構成例（開発者向け）

```toml
# .codex/config.toml（リポジトリルート）

approval_policy = "on-request"
sandbox_mode = "workspace-write"

[mcp_servers.context7]
command = "npx"
args = ["-y", "@upstash/context7-mcp"]

[mcp_servers.github]
command = "npx"
args = ["-y", "@modelcontextprotocol/server-github@latest"]
env_vars = ["GITHUB_TOKEN"]
default_tools_approval_mode = "prompt"
```

```md
# AGENTS.md（同リポジトリ）

## MCP usage
- Use GitHub MCP for issues/PRs; use git CLI for local diff.
- Do not paste tokens into prompts or tool args.
```

## この章の完了条件

- [ ] `[mcp_servers.*]` で STDIO / HTTP サーバーを定義できる
- [ ] `codex mcp add` と `/mcp` の使い方を説明できる
- [ ] ツール承認 `approval_mode` を設定できる
- [ ] MCP 設定と AGENTS.md の役割分担を説明できる
- [ ] `.codex/agents/` がサブエージェント定義であることを理解できる

次章では、GitHub Actions の **`openai/codex-action`** と AGENTS.md を組み合わせた CI 連携を扱います。
