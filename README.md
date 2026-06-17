# OpenAI Codex実践設計入門 Zenn Starter

Zenn向けの初期原稿セットです。

## 構成

```text
.
├── AGENTS.md
├── articles/
│   └── codex-book-teaser.md
├── books/
│   └── openai-codex-design-book/
│       ├── config.yaml
│       ├── why-codex-book.md
│       ├── codex-cli-setup.md
│       ├── agents-md-design.md
│       ├── sandbox-approval.md
│       ├── reading-zapabob-codex.md
│       ├── mcp-integration.md
│       └── ci-codex-action.md
└── examples/
    ├── AGENTS.md.*.md
    ├── global-vs-project.md
    └── .github/codex/ ...
```

## 使い方

```bash
npm install
npm run preview   # http://localhost:8000
npm run new:article
npm run new:book
```

GitHub: https://github.com/zapabob/codex-zenn-starter

Zenn連携: https://zenn.dev/dashboard/deploys から `codex-zenn-starter` を連携

## Codex PR レビュー（CI）

`.github/workflows/codex-pr-review.yml` は **ChatGPT 認証**（`CODEX_AUTH_JSON`）で PR レビューを投稿します。

**初回セットアップ:**

```powershell
# ローカルで ChatGPT ログイン済みであること（codex login status）
Get-Content -Raw "$env:USERPROFILE\.codex\auth.json" | gh secret set CODEX_AUTH_JSON -R zapabob/codex-zenn-starter --app actions
```

1. 上記で GitHub secret `CODEX_AUTH_JSON` を登録
2. PR を作成するか Actions → **Codex PR review** → **Run workflow**
3. PR に Codex レビューコメントが付く

> `auth.json` はパスワード同等。リポジトリにコミットしないこと。

## v0.1 方針

- `published: false` のまま下書きで詰める
- まずは `price: 0` で無料公開するか、低価格にする
- 無料チャプターとして導入、CLI導入、AGENTS.mdを開けておく
- 反応を見てMCP、CI/CD、フォーク追従、Windows releaseを追加する

## 中心メッセージ

Codexを起動する本ではなく、Codexが安全に働ける開発環境を作る本。
