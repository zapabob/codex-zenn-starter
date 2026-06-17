# OpenAI Codex実践設計入門 Zenn Starter

Zenn向けの初期原稿セットです。

## 構成

```text
.
├── articles/
│   └── codex-book-teaser.md
└── books/
    └── openai-codex-design-book/
        ├── config.yaml
        ├── why-codex-book.md
        ├── codex-cli-setup.md
        ├── agents-md-design.md
        ├── sandbox-approval.md
        └── reading-zapabob-codex.md
```

## 使い方

```bash
npm init --yes
npm install zenn-cli
npx zenn init
# この starter の articles/ と books/ をコピー
npx zenn preview
```

## v0.1 方針

- `published: false` のまま下書きで詰める
- まずは `price: 0` で無料公開するか、低価格にする
- 無料チャプターとして導入、CLI導入、AGENTS.mdを開けておく
- 反応を見てMCP、CI/CD、フォーク追従、Windows releaseを追加する

## 中心メッセージ

Codexを起動する本ではなく、Codexが安全に働ける開発環境を作る本。
