# 2026-06-18 MILSPEC規律と_docsトレーサビリティ記事{cursor}

**完了:** 2026-06-18

## ユーザー要望

AIエージェントコーディング時代の MILSPEC 風規律 + SE ベストプラクティス + AGENTS.md / `_docs/` 実装ログ命名規約のエンジニア向け記事を Zenn 公開。

## 仮説 → 検証

| # | 仮説 | 結果 |
|---|------|------|
| 1 | 前回 AGENTS-SOP 記事と被る | ✅ 役割分担（構成 vs 規律+ログ）で差別化 |
| 2 | 既存 `_docs/*.md` を例示できる | ✅ 7 件の実ログを参照 |
| 3 | repo AGENTS.md に既に _docs 言及あり | ✅ Rules 17 行目を引用 |

## 実装内容

- `articles/agents-md-docs-traceability-milspec.md`（published: true）
- テンプレ: 要望 / 仮説検証 / 変更 / 検証 / 次

## 検証

- frontmatter: title, emoji, topics, published
- git push → Zenn 連携

## 次

- 本書第3章から本記事へリンク（任意）
