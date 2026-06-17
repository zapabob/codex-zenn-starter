# 2026-06-17 Codex CI ChatGPT認証 {main}

**完了時刻:** 2026-06-17 01:27:45 UTC  
**ブランチ:** main / ci/test-codex-pr-review  
**PR:** https://github.com/zapabob/codex-zenn-starter/pull/1

## ユーザー要望

「CodexをAuthで」— CI を `OPENAI_API_KEY` ではなく **ChatGPT 認証**（`~/.codex/auth.json`）で動かす。

## 仮説 → 検証（CoT）

| # | 仮説 | 結果 |
|---|------|------|
| 1 | `codex-action` + `openai-api-key` 必須 | ✅ APIキー未設定で proxy 失敗（run 27658627825） |
| 2 | `auth.json` を secret 復元 + `codex exec review` で代替可 | ✅ ChatGPT auth で MCP/GitHub 連携レビュー成功 |
| 3 | `--base` と positional PROMPT は併用不可 | ✅ exit 2；AGENTS.md にレビュー形式を移した |
| 4 | sandbox `network_access=false` だと API 不通 | ✅ config.toml を `true` に変更 |
| 5 | GITHUB_OUTPUT heredoc は NUL 含む出力で壊れる | ✅ artifact 経由 + `tr -d '\0'` で解決 |
| 6 | GHA 上 bwrap が uid map で失敗することがある | 🔄 `--dangerously-bypass-approvals-and-sandbox` 追加（次 run 待ち） |

## 実装内容

1. **workflow** `.github/workflows/codex-pr-review.yml`
   - `openai/codex-action` → `@openai/codex` CLI + `codex exec review`
   - secret: `CODEX_AUTH_JSON`（`gh secret set` で登録済み）
   - `CODEX_HOME=.github/codex/home`
   - レビュー結果: artifact → `post_review` で PR コメント

2. **secret 登録**（PowerShell）
   ```powershell
   Get-Content -Raw "$env:USERPROFILE\.codex\auth.json" | gh secret set CODEX_AUTH_JSON -R zapabob/codex-zenn-starter --app actions
   ```

3. **ドキュメント**
   - 第7章 `ci-codex-action.md` — APIキー vs ChatGPT 認証の2経路
   - `AGENTS.md` — CI PR review 出力形式
   - `.gitignore` — `.github/codex/home/*` は config.toml のみ追跡

## CI 結果

| Run | 結果 | メモ |
|-----|------|------|
| 27658627825 | ❌ | OPENAI_API_KEY 未設定 |
| 27659212179 | ❌ | レビュー成功、GITHUB_OUTPUT NUL で失敗 |
| **27659451365** | **✅ success** | PR #1 にコメント投稿完了 |

## 注意

- `auth.json` はパスワード同等。公開 repo では API キー + codex-action を推奨。
- 誤って `.github/codex/home/` にローカル session を commit したが `4e5f4eb` で除去済み。

## 次

- sandbox bypass コミット後の PR 再レビューで diff 品質確認
- 第7章に artifact パターン追記（任意）
