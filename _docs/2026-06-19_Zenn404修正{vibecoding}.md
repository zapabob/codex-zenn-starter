# 2026-06-19 Zenn 404修正{vibecoding}

**完了:** 2026-06-19

## 症状

`leave-vibecoding-agent-coding` が 404。直前の `software-engineering-best-practices-agent-era` は公開済み。

## 仮説 → 検証

| # | 仮説 | 結果 |
|---|------|------|
| 1 | push 未反映 | ❌ remote `06cd44f` 確認済み |
| 2 | 非公式 `slug:` frontmatter でデプロイ全体失敗 | ✅ 削除（slug はファイル名のみ） |
| 3 | topics `vibecoding` | ✅ 除去済み |
| 3 | レートリミット | 要 Zenn ダッシュボード確認 |

## 対応

- 削除: `articles/leave-vibecoding-agent-coding.md`
- 追加: `articles/vibecoding-to-agent-coding.md`（`slug:` 明示、topics から `vibecoding` 除去、冒頭 blockquote 修正）
- 正 URL: https://zenn.dev/zapabob/articles/vibecoding-to-agent-coding

## 次

- デプロイ後 URL 確認
- 失敗時は https://zenn.dev/dashboard/deploys でエラー文言確認
