---
title: "Codex CLIを安全に導入する"
free: true
---

# Codex CLIを安全に導入する

Codex CLIは、OpenAIが公開しているターミナル上のコーディングエージェントです。Rust製で、選択したディレクトリ内のファイルを読み、編集し、シェルコマンドを実行しながら作業を進めます。

この章では「とりあえず起動する」だけでなく、**認証・作業ディレクトリ・サンドボックス・設定ファイル**まで含めた最小の安全構成を扱います。

## Codex CLIとIDE拡張の違い

Codexには複数の入口があります。

| 入口 | 用途 |
|------|------|
| `codex`（CLI） | ターミナルで対話しながらリポジトリを編集する |
| IDE拡張（VS Code / Cursor 等） | エディタ内でCodexを使う |
| `codex app` | デスクトップアプリ |
| Codex Web | ブラウザ上のクラウドエージェント |

この本の中心はCLIです。CLIは設定ファイル（`config.toml`）や `codex exec` による自動化、Sandbox / Approval の明示的な制御がしやすく、**開発環境を設計する**という本書のテーマに合います。

## インストール

### Windows（PowerShell）

公式の推奨インストール方法は次のとおりです。

```powershell
powershell -ExecutionPolicy ByPass -c "irm https://chatgpt.com/codex/install.ps1 | iex"
```

### npm / Homebrew

```powershell
npm install -g @openai/codex
```

```bash
brew install --cask codex
```

### バージョン確認

インストール後、まずバージョンを確認します。

```powershell
codex --version
```

CLIの機能やフラグ名はバージョンで変わることがあります。本書の例は [OpenAI Codex 公式ドキュメント](https://developers.openai.com/codex) を基準にしています。

## 認証

Codex CLIを初めて起動すると、認証方法を選びます。

```powershell
codex
```

対話UI（TUI）が開いたら、**Sign in with ChatGPT** を選ぶのが一般的です。ChatGPT Plus / Pro / Business / Edu / Enterprise プランにCodexが含まれる場合があります。

APIキーでの利用も可能ですが、設定手順は別途必要です。詳細は [Codex 認証ドキュメント](https://developers.openai.com/codex/auth) を参照してください。

:::message alert
認証情報は `~/.codex/` 配下に保存されます。`auth.json` はパスワードと同じ扱いで、リポジトリにコミットしないでください。
:::

## 作業を始める前の確認

CodexはGitリポジトリ内での作業を前提としています。作業前に、次を確認します。

```powershell
cd C:\path\to\your-repo
git status
git diff --stat
```

確認ポイントは次のとおりです。

- 作業ディレクトリが意図したリポジトリか
- 未コミットの変更を把握しているか
- `.env` や秘密鍵が作業対象に含まれていないか
- 失敗したときに `git checkout -- .` や `git stash` で戻せるか

Codexに任せる作業は、最初から小さく切ります。

- READMEの整備
- テストコマンドの確認
- 既存issueの調査
- 1ファイル限定の小さな修正
- AGENTS.mdの初期化

大規模リファクタリングや依存関係の一括更新は、SandboxとApprovalの設計が固まってからにします。

## 対話モードで起動する

リポジトリのルートで Codex を起動します。

```powershell
cd C:\path\to\your-repo
codex
```

TUI内では、次のような操作が使えます。

- 自然言語で作業を依頼する
- `/model` でモデルや推論レベルを切り替える
- `/permissions` でサンドボックスと承認モードを切り替える

計画だけしたいときは、`read-only` モードに切り替えてから依頼すると、ファイル変更やコマンド実行を抑えられます。

## サンドボックスと承認を明示する

Codex CLIは、デフォルトで **ネットワークアクセスがオフ** のサンドボックス内で動作します。ローカルではOSレベルのサンドボックスが、作業範囲（通常は現在のワークスペース）と書き込み権限を制限します。

### よく使う起動パターン

**Auto（日常開発の基本）** — ワークスペース内の読み書きとコマンド実行は自動、境界を越える操作は承認を求める:

```powershell
codex --sandbox workspace-write --ask-for-approval on-request
```

**読み取り専用で調査** — コードを読んで説明・計画だけさせる:

```powershell
codex --sandbox read-only --ask-for-approval on-request
```

**自動化（CI向け・要注意）** — 承認プロンプトなし。隔離された環境でのみ使う:

```powershell
codex exec --sandbox workspace-write --ask-for-approval never "summarize test failures"
```

**フルアクセス（原則非推奨）** — サンドボックスと承認をバイパス。使うなら使い捨てコンテナ等の隔離環境のみ:

```powershell
codex --dangerously-bypass-approvals-and-sandbox
```

:::message alert
`danger-full-access` と `approval_policy = "never"` の組み合わせは、意図しないファイル削除やネットワークアクセスにつながります。日常開発では使わないでください。
:::

### サンドボックスモード一覧

| モード | できること |
|--------|-----------|
| `read-only` | ファイル読み取りのみ。編集・シェル実行なし |
| `workspace-write` | ワークスペース内の読み書きとローカルコマンド（デフォルトの低摩擦モード） |
| `danger-full-access` | サンドボックスなし。隔離環境専用 |

### 承認ポリシー一覧

| ポリシー | 挙動 |
|---------|------|
| `untrusted` | 信頼されていないコマンドの前に確認 |
| `on-request` | サンドボックス境界を越える操作の前に確認 |
| `never` | 承認プロンプトなし（自動化専用） |

## 設定ファイル `config.toml`

CLIフラグを毎回打たずに済むよう、ユーザー設定を `~/.codex/config.toml` に書きます。

```toml
# ~/.codex/config.toml

model = "gpt-5.3-codex"
approval_policy = "on-request"
sandbox_mode = "workspace-write"

# AGENTS.md をプロジェクト文書として読む（デフォルトで有効）
project_doc_fallback_filenames = ["AGENTS.md"]
project_doc_max_bytes = 32768

[sandbox_workspace_write]
network_access = false

[windows]
sandbox = "elevated"   # 推奨。管理者セットアップが必要
# sandbox = "unelevated" # elevated が使えない環境のフォallback
```

設定の優先順位（上ほど強い）:

1. CLIフラグと `--config` 上書き
2. プロジェクトの `.codex/config.toml`（リポジトリルートから cwd まで、近い方が優先）
3. プロファイル `~/.codex/<profile>.config.toml`（`--profile` 指定時）
4. ユーザー設定 `~/.codex/config.toml`
5. 組み込みデフォルト

プロジェクトごとにルールを変えたい場合は、リポジトリに `.codex/config.toml` を置きます。

```toml
# your-repo/.codex/config.toml

approval_policy = "on-request"
sandbox_mode = "workspace-write"
```

## Windows での注意点

Windows 11 が推奨環境です。Codexは **ネイティブWindows** と **WSL2** の両方で動きます。

### ネイティブ Windows サンドボックス

PowerShell から直接 `codex` を動かす場合、Windows 専用サンドボックスが使われます。

- `elevated` — 低権限サンドボックスユーザー、ファイアウォール、ACL による境界（推奨）
- `unelevated` — 管理者権限不要のフォールバック。境界は弱い

セットアップに失敗した場合、Codexは `unelevated` にフォールバックすることがあります。ログは `%USERPROFILE%\.codex\.sandbox\sandbox.log` に出力されます。

サンドボックスが特定ディレクトリを読めないときは、セッション中に次を使います。

```text
/sandbox-add-read-dir C:\absolute\directory\path
```

### WSL2 を使う場合

Linux ネイティブのツールチェーンが必要なとき、または `/home/...` 配下で高速I/Oが欲しいときは WSL2 が向いています。

```powershell
wsl --install
wsl
```

```bash
curl -fsSL https://chatgpt.com/codex/install.sh | sh
cd ~/code/your-repo
codex
```

:::message
WSL 内では `/mnt/c/...` より `~/code/...` の方が速く、権限トラブルも少ないです。
:::

### この本の方針

| 状況 | 推奨 |
|------|------|
| Windows 11 + PowerShell 日常開発 | ネイティブ `elevated` サンドボックス |
| Rust / Linux ツールが前提 | WSL2 |
| CI | Linux ランナー + `codex exec` |

## 非対話モード `codex exec`

スクリプトやCIから Codex を呼ぶときは `codex exec` を使います。

```powershell
codex exec "リポジトリ構成を要約し、テスト実行方法を特定してください"
```

デフォルトは **read-only サンドボックス** です。編集を許す場合は明示します。

```powershell
codex exec --sandbox workspace-write "README にセットアップ手順を追記してください"
```

パイプとの組み合わせ例:

```powershell
git diff | codex exec "この差分のリスクを3点で指摘してください"
```

JSON Lines 出力（自動処理向け）:

```powershell
codex exec --json "テスト失敗の原因を特定してください" | Out-File result.jsonl
```

:::details 認証（CI）
GitHub Actions では [openai/codex-action](https://github.com/openai/codex-action) の利用が推奨されています。`OPENAI_API_KEY` をジョブ全体の環境変数に置かず、Action 経由で渡すことで、リポジトリ内の任意コードからキーが読まれるリスクを下げられます。
:::

## 最初の安全なセッション例

以下は、新しいリポジトリで Codex を試すときの手順例です。

```powershell
# 1. リポジトリへ移動
cd C:\Users\you\Projects\my-app

# 2. Git 状態を確認
git status

# 3. 読み取り専用で調査だけ依頼
codex --sandbox read-only --ask-for-approval on-request

# TUI 内で:
# 「このリポジトリのテスト実行方法と主要ディレクトリを説明してください」

# 4. 小さな編集に切り替え
# /permissions で workspace-write + on-request に変更

# 「README に test コマンドの節を追加してください。変更後にテストを実行してください」
```

## この章の完了条件

この章を読み終えた時点で、次ができる状態を目指します。

- [ ] Codex CLI をインストールし、`codex --version` で確認できる
- [ ] ChatGPT または API キーで認証できる
- [ ] Git 管理されたリポジトリで `codex` を起動できる
- [ ] `--sandbox` と `--ask-for-approval` の意味を説明できる
- [ ] `~/.codex/config.toml` に基本設定を書ける
- [ ] Windows では `elevated` / WSL2 の使い分けを判断できる
- [ ] `codex exec` で非対話実行の最小例を書ける

次章では、Codex がリポジトリ内のルールを読む **`AGENTS.md` の設計** に進みます。
