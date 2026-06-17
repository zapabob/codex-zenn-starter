# AGENTS.md テンプレート集

このディレクトリは、[OpenAI Codex実践設計入門](https://zenn.dev/zapabob/books/openai-codex-design-book) 用の **リポジトリ向け AGENTS.md テンプレート** です。

## 使い方

1. スタックに近いテンプレートを選ぶ
2. リポジトリルートに `AGENTS.md` としてコピー
3. `Commands` / `Rules` / `Done` をプロジェクトに合わせて編集
4. 必要なら `.codex/config.toml` も同梱（第6章・第7章参照）

```powershell
Copy-Item examples\AGENTS.md.python-cli.md .\AGENTS.md
```

## テンプレート一覧

| ファイル | 想定スタック |
|---------|-------------|
| `AGENTS.md.python-cli.md` | Python + uv + pytest + ruff |
| `AGENTS.md.node-nextjs.md` | Node.js / Next.js + npm |
| `AGENTS.md.rust-cli.md` | Rust CLI / cargo workspace |
| `AGENTS.md.codex-fork.md` | openai/codex 系 upstream-first フォーク |
| `AGENTS.md.with-sop-design.md` | SOP / DESIGN.md 参照付き AGENTS.md |
| `docs/sop/engineering.md` | 共通エンジニアリング SOP |
| `docs/design/DESIGN.md` | フロントエンド設計例 |
| `global-vs-project.md` | `~/.codex/AGENTS.md` とリポジトリ `AGENTS.md` の違い |

## CI 向けサンプル

| パス | 内容 |
|------|------|
| `.github/codex/home/config.toml` | CI 用 Codex 設定（安全な sandbox） |
| `.github/codex/prompts/review.md` | PR レビュー用プロンプト |
| `.github/workflows/codex-pr-review.yml` | ChatGPT 認証（`CODEX_AUTH_JSON`）で PR レビュー |

## 注意

- **`~/.codex/AGENTS.md` はユーザー全体の設定** であり、リポジトリにコミットしません（`global-vs-project.md` 参照）
- テンプレート内の API キー・トークンはすべてプレースホルダです。実値をコミットしないでください
