---
title: "GitHub ActionsとCIにCodexを組み込む"
free: true
---

# GitHub ActionsとCIにCodexを組み込む

Codex を日常開発に入れた次の段階は、**CI/CD に組み込む** ことです。

PR レビュー、テスト失敗時の修正案生成、リリースノート作成など、反復的な判断を Codex に任せられます。

この章では [openai/codex-action](https://github.com/openai/codex-action) を使い、**AGENTS.md と安全な sandbox 設定** を CI に持ち込む方法を扱います。

## 認証方式: API キー vs ChatGPT

CI では次の2通りがあります。

| 方式 | secret | 向いているケース |
|------|--------|-----------------|
| **API キー** | `OPENAI_API_KEY` | 公開リポジトリ、チーム共有、公式 `codex-action` |
| **ChatGPT** | `CODEX_AUTH_JSON` | 個人/非公開リポ、ローカルと同じ ChatGPT プランで回したい |

ローカルで `codex login status` が **Logged in using ChatGPT** なら、`~/.codex/auth.json` を GitHub secret に登録できます。

```powershell
gh secret set CODEX_AUTH_JSON -R owner/repo --app actions < "$env:USERPROFILE\.codex\auth.json"
```

:::message alert
`auth.json` はパスワード同等です。リポジトリにコミットしないでください。公開リポジトリでは **API キー + codex-action** を推奨します。
:::

## ChatGPT 認証で PR レビュー（codex exec review）

[codex-zenn-starter](https://github.com/zapabob/codex-zenn-starter) では **ChatGPT 認証** を使い、`codex exec review` で PR diff をレビューします（`codex-action` の proxy は使いません）。

```yaml
      - uses: actions/setup-node@v4
        with:
          node-version: "20"

      - name: Install Codex CLI
        run: npm install -g @openai/codex

      - name: Restore Codex ChatGPT auth
        run: |
          mkdir -p .github/codex/home
          printf '%s' "$CODEX_AUTH_JSON" > .github/codex/home/auth.json
          chmod 600 .github/codex/home/auth.json
        env:
          CODEX_AUTH_JSON: ${{ secrets.CODEX_AUTH_JSON }}

      - name: Run Codex review (ChatGPT auth)
        id: run_codex
        env:
          CODEX_HOME: ${{ github.workspace }}/.github/codex/home
        run: |
          codex exec review \
            --base "${{ github.event.pull_request.base.ref }}" \
            -o codex-review.md
```

:::message
`codex exec review --base` はカスタム PROMPT 引数と併用できません。レビュー観点は **AGENTS.md** に書き、Codex が自動読み込みします。`prompts/review.md` は `codex exec`（非 review）や codex-action 向けの参考です。
:::

## なぜ codex-action を使うか（API キー経路）

CI から `npx @openai/codex` を自分で入れ、`OPENAI_API_KEY` を export する方法もあります。

GitHub Actions では **`openai/codex-action@v1`** が API キー利用時に推奨されます。

| 機能 | 自前 install | codex-action |
|------|-------------|--------------|
| CLI インストール | 自分で管理 | Action が処理 |
| API キー保護 | ジョブ全体に export しがち | proxy + safety-strategy |
| sudo 降格 | 自分で実装 | `drop-sudo` 組み込み |
| 出力取得 | 自分でパース | `final-message` output |

:::message alert
`OPENAI_API_KEY` を **ジョブレベルの `env:`** に置かないでください。checkout 後に動く任意のスクリプトや依存の lifecycle hook から読まれる可能性があります。Action の `openai-api-key` 入力経由で渡します。
:::

## ディレクトリ構成（推奨）

```
your-repo/
├── AGENTS.md                          # Codex が読むリポジトリルール
├── .github/
│   ├── codex/
│   │   ├── home/
│   │   │   └── config.toml            # CI 専用 Codex 設定
│   │   └── prompts/
│   │       └── review.md              # タスク別プロンプト
│   └── workflows/
│       └── codex-pr-review.yml
└── examples/                          # 本リポジトリ同梱テンプレ参照可
```

`codex-home: .github/codex/home` を指定すると、CI 実行時だけ **安全な config.toml** が使われます。開発者の `~/.codex/config.toml`（緩い設定）が CI に漏れません。

## CI 用 config.toml

`examples/.github/codex/home/config.toml`:

```toml
approval_policy = "never"
sandbox_mode = "workspace-write"

project_doc_fallback_filenames = ["AGENTS.md"]

[sandbox_workspace_write]
network_access = false
```

CI では承認プロンプトが出せないため `approval_policy = "never"` になります。その代わり **sandbox を workspace-write に固定** し、ネットワークはオフにします。

`danger-full-access` を CI に持ち込まないでください。

## PR レビュー用プロンプト

`examples/.github/codex/prompts/review.md`:

```md
Review this pull request and respond with a concise Markdown message.

Structure:
1. **Summary** — 1–2 sentences on what changed
2. **Review** — risks, test gaps, or suggestions

Constraints:
- Follow AGENTS.md in this repository for test/lint expectations.
- Do not suggest unrelated refactors.
```

公式 openai/codex リポジトリにも `.github/codex/labels/codex-review.md` のような短いプロンプト例があります。プロンプトは **リポジトリにコミット** し、レビュー可能にします。

## ワークフロー全体像

`examples/.github/workflows/codex-pr-review.yml` をベースにします。

```yaml
name: Codex PR review

on:
  pull_request:
    types: [opened, synchronize, reopened]

jobs:
  codex_review:
    runs-on: ubuntu-latest
    permissions:
      contents: read
    outputs:
      final_message: ${{ steps.run_codex.outputs.final-message }}
    steps:
      - uses: actions/checkout@v5
        with:
          ref: refs/pull/${{ github.event.pull_request.number }}/merge
          persist-credentials: false

      - name: Pre-fetch PR refs
        env:
          PR_BASE_REF: ${{ github.event.pull_request.base.ref }}
          PR_NUMBER: ${{ github.event.pull_request.number }}
        run: |
          git fetch --no-tags origin \
            "$PR_BASE_REF" \
            "+refs/pull/$PR_NUMBER/head"

      - name: Run Codex
        id: run_codex
        uses: openai/codex-action@v1
        with:
          openai-api-key: ${{ secrets.OPENAI_API_KEY }}
          prompt-file: .github/codex/prompts/review.md
          sandbox: workspace-write
          safety-strategy: drop-sudo
          codex-home: .github/codex/home
          output-file: codex-review.md
```

### 重要な入力

| 入力 | 用途 |
|------|------|
| `openai-api-key` | API キー（secrets から） |
| `prompt` / `prompt-file` | タスク指示（どちらか一方のみ） |
| `sandbox` | `read-only` / `workspace-write` |
| `safety-strategy: drop-sudo` | sudo 降格でメモリ内 secret リスク低減 |
| `codex-home` | CI 専用 config.toml の場所 |
| `codex-args` | 追加 CLI フラグ（JSON 配列 or シェル文字列） |
| `output-file` | 最終メッセージの保存 |

### コメント投稿ジョブ

Codex ジョブは `contents: read` のみ。パッチ適用や PR コメントは **別ジョブ** で `pull-requests: write` を付与します。

```yaml
  post_review:
    needs: codex_review
    permissions:
      pull-requests: write
    steps:
      - uses: actions/github-script@v7
        with:
          script: |
            await github.rest.issues.createComment({ ... });
        env:
          CODEX_FINAL_MESSAGE: ${{ needs.codex_review.outputs.final_message }}
```

## AGENTS.md 連携の要点

Codex Action は checkout したリポジトリ内の **AGENTS.md を自動読み込み** します（`project_doc_fallback_filenames` デフォルト）。

CI プロンプトと AGENTS.md の役割分担:

| ファイル | 書く内容 |
|---------|---------|
| `AGENTS.md` | test / lint コマンド、禁止事項、完了条件 |
| `.github/codex/prompts/*.md` | その CI ジョブ固有の出力形式・観点 |

悪い例: プロンプトに毎回 `npm test` を書く  
よい例: AGENTS.md に `npm test` を書き、プロンプトは「AGENTS.md に従え」と参照する

## テスト失敗時の自動修正パターン

公式ドキュメントの [Non-interactive mode](https://developers.openai.com/codex/noninteractive) より、CI 失敗後に Codex で patch を生成し、**別ジョブで PR を開く** パターンが紹介されています。

要点:

1. `workflow_run` で失敗 CI をトリガー
2. Codex ジョブは `contents: read` + `drop-sudo`
3. `git diff` を artifact 化
4. `open_pr` ジョブだけ write 権限

Codex が patch を書いても、**人間レビューなしに main へ merge しない** 運用にします。

## Windows ランナー

codex-action は Linux / macOS が前提です。Windows ランナーでは `safety-strategy: unsafe` が必要になり、secret 保護が弱くなります。

Windows 中心のリポジトリでも、Codex CI ジョブは **`ubuntu-latest`** で動かすのが無難です。

## セキュリティチェックリスト

- [ ] `OPENAI_API_KEY` は secrets 経由のみ。ジョブ `env:` に置かない
- [ ] `safety-strategy: drop-sudo`（Linux/macOS）
- [ ] Codex ステップをジョブの **最後** に近い位置に（後続ステップが状態を継承しない）
- [ ] PR 本文・コメントからの **プロンプトインジェクション** を sanitize
- [ ] `allow-users` / `allow-bots` で workflow トリガーを制限
- [ ] CI config は `workspace-write` + `network_access = false`
- [ ] 自動 patch は必ず PR 経由。直接 push しない

## 本リポジトリへの適用

[codex-zenn-starter](https://github.com/zapabob/codex-zenn-starter) では:

1. `examples/.github/` を `.github/` にコピー
2. `secrets.CODEX_AUTH_JSON` にローカルの `~/.codex/auth.json` を登録（ChatGPT 認証）
3. PR 時に Zenn 原稿の Markdown 品質レビューを Codex に任せる

API キー経路を使う場合は `OPENAI_API_KEY` を登録し、workflow を `openai/codex-action@v1` 版に差し替えてください。

Zenn 原稿向け AGENTS.md 例:

```md
## Commands
- Preview: `npm run preview`
- Lint: n/a (Markdown manual review)

## Rules
- Do not set `published: true` unless release is intended.
- Keep reader-facing tone; no author-internal notes.
```

## この章の完了条件

- [ ] ChatGPT 認証（`CODEX_AUTH_JSON`）か API キー（`codex-action`）のどちらを使うか決められる
- [ ] `codex-home` で CI 専用 config を分離できる
- [ ] AGENTS.md と prompt-file の役割分担を説明できる
- [ ] `drop-sudo` と sandbox 設定の理由を説明できる
- [ ] レビュー用と autofix 用のジョブ権限分離を設計できる

---

本 v0.2 時点で、MCP（第6章）と CI（第7章）までをカバーしました。`examples/` ディレクトリのテンプレートをコピーして、自リポジトリに AGENTS.md と GitHub Actions を展開してください。
