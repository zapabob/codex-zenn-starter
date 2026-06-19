# 2026-06-19 Zenn 404修正{vibecoding}

**完了:** 2026-06-19

## 症状

`leave-vibecoding-agent-coding` が 404。直前の `software-engineering-best-practices-agent-era` は公開済み。

## 原因（確定）

**Zenn GitHub 連携のレートリミット。** ダッシュボードで確認。10:32〜10:43 に記事を4本連続 push したタイミングで、最新1本のデプロイがスキップ／失敗した。

404 ≠ 記事ファイルが無い。= **デプロイがキューに載らなかった／落ちた** パターン。

## 実施した修正（予防）

| 項目 | 旧 | 新 |
|------|----|----|
| ファイル名 | `leave-vibecoding-agent-coding.md` | `vibecoding-to-agent-coding.md` |
| topics | `vibecoding` 含む | 既存記事と同型（`antipattern` 等） |
| 比較表 | 空ヘッダ列 | `観点` 列を明示 |
| 非公式 frontmatter | `slug:` 追加（誤） | 削除（slug はファイル名のみ） |

## 正 URL

https://zenn.dev/zapabob/articles/vibecoding-to-agent-coding

## 運用ルール（再発防止）

1. **記事は1本 push ごとに 15〜30 分空ける**（連続4本はアウト）
2. 下書きは `published: false` のまま repo に置き、公開は1本ずつ
3. 404 時はまず https://zenn.dev/dashboard/deploys で「Rate limit」を確認
4. リミット解除後、**1 commit だけ** push して反映待ち（5〜10分）

## 次

- レートリミット解除後、全文入り `vibecoding-to-agent-coding.md` を push → URL 200 確認
