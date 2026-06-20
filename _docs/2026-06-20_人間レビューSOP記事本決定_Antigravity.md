# 人間レビューSOP記事本決定 実装ログ

## Overview

AI時代の人間レビューのボトルネックについて、SOP（標準作業手順）を用いた設計アプローチを提案するZenn記事『人間は何をレビューすべきか？ AI時代のコードレビューをSOPで設計し直す』の正式稿を作成し、不要となった叩き台用の空ファイルをクリーンアップしました。

## Background / requirements

- ユーザーのフィードバックに基づき、3案目をベースにして、Zenn向けに少し落ち着かせたタイトル「人間は何をレビューすべきか？ AI時代のコードレビューをSOPで設計し直す」を採用。
- ZennのFront Matterフォーマット（title, emoji, type, topics, published）を正しく設定する。
- OpenAI公式ドキュメントでの位置づけを考慮し、`codex exec review` のような記述を「Codexの `/review` や GitHub Actions上のAIレビュー」のように表現を調整する。
- 以前の叩き台作成時に作成された `articles/` 配下の不要な空ファイルを削除してリポジトリをクリーンにする。
- `zenn preview` が正常に動作することを確認する。

## Assumptions / decisions

- 記事ファイルとして `articles/what-should-humans-review.md` を使用。
- 不要な空ファイル `articles/ai-era-code-review-sop.md`, `articles/what-should-humans-review-sop.md`, `articles/what-to-review-ai-sop.md` は削除。
- 依存関係のインストールおよびプレビューは、プロジェクト規則に従い `pnpm` を使用。

## Changed files

- [MODIFY] [what-should-humans-review.md](file:///c:/Users/downl/Downloads/codex-zenn-starter/articles/what-should-humans-review.md) - Zenn向け正式稿（Front Matterおよび本文）の書き込み。
- [DELETE] `articles/ai-era-code-review-sop.md`
- [DELETE] `articles/what-should-humans-review-sop.md`
- [DELETE] `articles/what-to-review-ai-sop.md`

## Implementation details

- フロントマターは以下のように定義：
  ```yaml
  ---
  title: "人間は何をレビューすべきか？ AI時代のコードレビューをSOPで設計し直す"
  emoji: "🧭"
  type: "tech"
  topics: ["codex", "aiagent", "codereview", "githubactions", "sop"]
  published: false
  ---
  ```
- 本文中のAIレビューやCIに関する記述は、OpenAI公式に準じた安全な表現（「Codexの `/review` や GitHub Actions上のAIレビュー」など）を反映し、具体的な「codex exec review」などの未サポート/誤解を招く記述を排除。

## Commands run

- `pnpm install` - Zenn CLIなどの依存関係をpnpmでインストール
- `Remove-Item articles/ai-era-code-review-sop.md, articles/what-should-humans-review-sop.md, articles/what-to-review-ai-sop.md` - 空ファイルの削除
- `pnpm run preview` - Zenn CLIによる記事プレビューの動作確認

## Test / verification results

- `pnpm run preview` 実行により `http://localhost:8000` でプレビューサーバーがエラーなしで立ち上がることを検証。
- バリデーションエラーなどが発生しないことを確認してプレビューサーバーを停止。

## Residual risks

- 記事ファイルは `published: false` に設定されているため、マージ・デプロイされてもZenn上には公開されません（ドラフト扱い）。本リリース時には `published: true` に設定変更する必要があります。

## Recommended next actions

- プレビュー上で最終的な表示を確認し、内容に問題がなければ `published: true` へ切り替えて公開、またはGitHub統合による自動デプロイを行ってください。
