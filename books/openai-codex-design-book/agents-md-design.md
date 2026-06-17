---
title: "AGENTS.mdでリポジトリに規律を与える"
free: true
---

# AGENTS.mdでリポジトリに規律を与える

AGENTS.mdは、CodexのようなAIコーディングエージェントに対して「このリポジトリではどう作業してほしいか」を伝えるファイルです。

READMEが人間向けのプロジェクト説明だとすれば、AGENTS.mdは **AI向けの開発ルール** です。Codex CLIは起動時にこのファイルを読み込み、作業の前提として使います。

## Codex が AGENTS.md を読む仕組み

Codex CLIの設定キー `project_doc_fallback_filenames` のデフォルトは `["AGENTS.md"]` です。リポジトリルート（または cwd からルート方向）にある AGENTS.md が、プロジェクト文書として自動注入されます。

上限サイズは `project_doc_max_bytes`（デフォルト 32 KiB）です。長すぎると切り詰められるため、**実行可能なルールに絞る** のが重要です。

設定例:

```toml
# ~/.codex/config.toml または .codex/config.toml
project_doc_fallback_filenames = ["AGENTS.md", "CLAUDE.md"]
project_doc_max_bytes = 32768
```

複数ファイルを列挙できますが、本書では **AGENTS.md を正本** とし、他ファイルは移行期間のエイリアス程度に留めることを推奨します。

## AGENTS.md に書くこと

最低限、次の項目を書きます。

| セクション | 内容 |
|-----------|------|
| Project | リポジトリの目的と技術スタック |
| Commands | install / test / lint / format の **実行可能なコマンド** |
| Rules | 変更してよい場所、禁止事項、secrets の扱い |
| Done | 完了条件（テスト通過、lint 通過など） |

抽象的な精神論より、Codex がそのまま実行できる指示を優先します。

## 悪い AGENTS.md の例

```md
# AGENTS.md

いい感じに実装してください。
テストも適当にお願いします。
```

Codexは判断基準を持てません。「いい感じ」「適当」は再現性のない指示です。

## よい AGENTS.md の最小例（Python）

```md
# AGENTS.md

## Project
Python CLI application managed with uv.

## Commands
- Install: `uv sync`
- Test: `uv run pytest`
- Lint: `uv run ruff check .`
- Format: `uv run ruff format .`

## Rules
- Do not read or print secrets from `.env`.
- Do not modify files under `data/raw/`.
- Keep changes small; run tests after every edit.

## Done
- `uv run pytest` passes.
- `uv run ruff check .` passes.
- Final response lists changed files and verification commands run.
```

## よい AGENTS.md の最小例（Node.js / TypeScript）

```md
# AGENTS.md

## Project
Next.js 15 app with npm workspaces.

## Commands
- Install: `npm ci`
- Dev: `npm run dev`
- Test: `npm test`
- Lint: `npm run lint`
- Typecheck: `npm run typecheck`

## Rules
- Do not commit `.env.local`.
- Do not edit generated files under `.next/`.
- Prefer existing components in `src/components/`.

## Done
- `npm test` and `npm run lint` pass.
- No new TypeScript errors (`npm run typecheck`).
```

## 実例: openai/codex の AGENTS.md

公式 [openai/codex](https://github.com/openai/codex) リポジトリ自身が、大規模 Rust モノレポ向け AGENTS.md の実例です。冒頭は次のような **具体的な開発規約** から始まります。

- クレート名は `codex-` プレフィックス
- `format!` では可能な限り変数を `{}` にインライン
- テスト前に `just`、`rg`、`cargo-insta` 等の依存コマンドを確認
- サンドボックス関連の環境変数 `CODEX_SANDBOX_*` を安易に変更しない

大規模リポジトリでは、「何を実行するか」だけでなく **「何を触ってはいけないか」** まで AGENTS.md に書かれています。Codex 本体の開発リポジトリが、AGENTS.md のベストプラクティスそのものと言えます。

## プロジェクト設定との組み合わせ

AGENTS.md は「何をしてほしいか」、`.codex/config.toml` は「どこまで許可するか」を分担します。

```
your-repo/
├── AGENTS.md              # 作業ルール（AI向け）
├── .codex/
│   └── config.toml        # Sandbox / Approval のデフォルト
├── README.md              # 人間向け概要
└── src/
```

プロジェクトの `.codex/config.toml` 例:

```toml
approval_policy = "on-request"
sandbox_mode = "workspace-write"

[sandbox_workspace_write]
network_access = false
```

AGENTS.md に「ネットワークアクセスは禁止」と書き、config でも `network_access = false` にする。**文書と設定の二重化** は、どちらか一方が漏れても安全側に倒れるため有効です。

## Codex にとって読みやすい書き方

### 実行可能なルールにする

| 悪い例 | よい例 |
|--------|--------|
| テストをちゃんと書いて | Test: `uv run pytest tests/` |
| セキュリティに注意 | Do not log or print `.env` contents |
| リファクタリングして | Refactor only `src/parser/`; run tests after each change |

### 完了条件を明文化する

Codex への依頼は「終わり方」が曖昧だと、テスト未実行のまま終了することがあります。

```md
## Done
- All tests pass: `cargo test -p codex-core`
- No clippy warnings: `cargo clippy -p codex-core -- -D warnings`
- Response includes: changed files, commands run, test output summary
```

### 禁止事項は否定形で

```md
## Rules
- Do NOT modify `Cargo.lock` unless adding a dependency.
- Do NOT run `git push`.
- Do NOT disable sandbox or approval settings.
```

## AGENTS.md テンプレート（zapabob/codex フォーク向け）

[zapabob/codex](https://github.com/zapabob/codex) のような upstream-first フォークでは、次の項目を追加します。

```md
# AGENTS.md

## Project
Fork of openai/codex. Upstream-first: prefer merging official changes before local extensions.

## Commands
- Build (Rust): `cargo build -p codex-cli` (from `codex-rs/`)
- Test: `cargo test -p codex-core`
- Upstream sync: `python scripts/upstream_sync.py` (maintainers only)

## Rules
- Do not modify sandbox env vars (`CODEX_SANDBOX_*`) in tests without understanding seatbelt/bubblewrap behavior.
- Extension code belongs under `plugins/`, `extensions/`, or `zapabob/` — not inline in upstream core.
- Run `just write-config-schema` after changing `ConfigToml`.

## Upstream
- Official baseline: https://github.com/openai/codex
- Sync driver: `scripts/upstream_sync.py`
- Changelog: `CHANGELOG.md`

## Done
- Tests pass for affected crates.
- No unintended diffs in upstream-owned directories.
```

## 段階的に育てる

最初から完璧な AGENTS.md は不要です。次の順で育てます。

1. **v0** — test / lint コマンドと `.env` 禁止だけ書く
2. **v1** — ディレクトリ構成と変更可能範囲を追加
3. **v2** — Done 条件、CI コマンド、Windows/WSL の差分を追記
4. **v3** — フォーク運用、MCP サーバー、プロファイル名など環境固有の節を追加

Codex 自身に AGENTS.md の下書きを作らせるのも有効です。

```powershell
codex --sandbox workspace-write --ask-for-approval on-request
```

```
このリポジトリを調査し、AGENTS.md の v0 ドラフトを作成してください。
test/lint コマンド、禁止事項、完了条件だけ含めてください。
```

人間がレビューしてからコミットします。AI が書いたルールを、そのまま無検証で merge しないでください。

## examples/ テンプレート一覧

[codex-zenn-starter](https://github.com/zapabob/codex-zenn-starter) の `examples/` には、スタック別の AGENTS.md を同梱しています。

| ファイル | 想定スタック |
|---------|-------------|
| `AGENTS.md.python-cli.md` | Python + uv + pytest |
| `AGENTS.md.node-nextjs.md` | Node.js / Next.js |
| `AGENTS.md.rust-cli.md` | Rust CLI / cargo |
| `AGENTS.md.codex-fork.md` | upstream-first フォーク |
| `AGENTS.md.with-mcp.md` | MCP 利用時の追加ルール |
| `global-vs-project.md` | `~/.codex/AGENTS.md` とリポジトリ `AGENTS.md` の違い |

```powershell
Copy-Item examples\AGENTS.md.python-cli.md .\AGENTS.md
```

## この章の完了条件

- [ ] 自リポジトリに AGENTS.md を置き、Codex 起動時に読まれることを確認できる
- [ ] Commands / Rules / Done の3セクションを書ける
- [ ] `.codex/config.toml` と役割分担できる
- [ ] 公式 openai/codex の AGENTS.md を参考に、大規模リポジトリ向けの書き方を理解できる
- [ ] `examples/` からスタックに合うテンプレを選べる

次章では、AGENTS.md だけでは足りない **Sandbox と Approval の権限設計** を扱います。
