---
title: "SandboxとApprovalを設計する"
free: true
---

# SandboxとApprovalを設計する

Codexを安全に使ううえで、いちばん軽視してはいけないのが **権限設計** です。

Codexはファイルを読み、変更し、コマンドを実行できます。便利であるほど、誤操作や情報漏えいの影響も大きくなります。

この章では、Sandbox（どこまで作業を許すか）と Approval（どこで人間の確認を挟むか）を、実務の作業単位と結びつけて設計します。

## Sandbox と Approval は別のレイヤー

| レイヤー | 役割 | 設定キー / フラグ |
|---------|------|------------------|
| Sandbox | ファイルシステム・ネットワークの **物理的な境界** | `sandbox_mode` / `--sandbox` |
| Approval | 境界を越える操作の前に **止める** | `approval_policy` / `--ask-for-approval` |

Sandbox だけ厳しくても Approval が `never` なら、サンドボックス内では無確認で破壊的操作が走ります。  
Approval だけ厳しくても Sandbox が `danger-full-access` なら、承認後にシステム全体へアクセスできます。

**両方をセットで設計する** のが基本です。

## サンドボックスモード

### read-only

- ファイル読み取りと質問への回答のみ
- 編集・シェル実行・ネットワークなし
- 用途: コード調査、設計レビュー、計画立案

```powershell
codex --sandbox read-only --ask-for-approval on-request
```

TUI 内では `/permissions` から `read-only` に切り替え可能です。

### workspace-write（日常開発のデフォルト）

- ワークスペース内の読み書き
- ワークスペース内でのローカルコマンド実行
- ネットワークはデフォルト **オフ**（明示的に有効化が必要）
- 用途: 通常の機能追加、テスト修正、ドキュメント更新

```powershell
codex --sandbox workspace-write --ask-for-approval on-request
```

`.git/` や `.codex/` など **保護パス** は、モードが `workspace-write` でも書き込み制限される場合があります。意図的な安全装置です。

### danger-full-access

- サンドボックス境界なし
- 用途: 使い捨て VM / コンテナ内の CI のみ

```toml
# 日常開発では書かない
sandbox_mode = "danger-full-access"
approval_policy = "never"
```

## 承認ポリシー

### untrusted

信頼されていないコマンドの実行前に確認します。Codex のデフォルトに近い保守的な設定です。

```powershell
codex --sandbox workspace-write --ask-for-approval untrusted
```

### on-request

サンドボックス内では自動実行し、**境界を越える操作**（ワークスペース外の編集、ネットワークアクセス等）の前に確認します。Auto プリセットの核心です。

```powershell
codex --sandbox workspace-write --ask-for-approval on-request
```

### never

承認プロンプトなし。**CI や完全隔離環境専用** です。

```powershell
codex exec --sandbox workspace-write --ask-for-approval never "fix lint errors"
```

:::message alert
`never` は「Codex を信頼する」設定ではなく、「人間がプロンプトで止められない」設定です。ローカル対話開発では使わないでください。
:::

## よく使う組み合わせ

| 意図 | sandbox | approval | コマンド例 |
|------|---------|----------|-----------|
| 調査・計画 | `read-only` | `on-request` | `codex --sandbox read-only --ask-for-approval on-request` |
| 日常開発（Auto） | `workspace-write` | `on-request` | フラグ省略 or 上記 |
| 編集は自動、未知コマンドだけ確認 | `workspace-write` | `untrusted` | `--ask-for-approval untrusted` |
| CI 自動修正 | `workspace-write` | `never` | `codex exec --sandbox workspace-write --ask-for-approval never "..."` |
| 隔離コンテナ内フル | `danger-full-access` | `never` | 非推奨・例外のみ |

## config.toml で固定する

毎回フラグを打たないよう、プロファイルとして固定します。

```toml
# ~/.codex/config.toml

# 日常開発プロファイル（デフォルト）
approval_policy = "on-request"
sandbox_mode = "workspace-write"

[sandbox_workspace_write]
network_access = false
# writable_roots = ["/extra/path"]  # 追加書き込み先が必要な場合

[windows]
sandbox = "elevated"
```

調査専用プロファイル:

```toml
# ~/.codex/research.config.toml
approval_policy = "on-request"
sandbox_mode = "read-only"
```

```powershell
codex --profile research
```

## ネットワークアクセス

デフォルトでは Codex のサンドボックス内 **ネットワークはオフ** です。パッケージインストールや API 呼び出しが必要なときは、次のいずれかが必要です。

1. セッション中の承認プロンプトで許可する
2. `config.toml` で明示的に有効化する

```toml
[sandbox_workspace_write]
network_access = true
```

ネットワークを常時オンにするのではなく、**必要な作業だけ一時的に許可** する運用が安全です。

## 追加ディレクトリへのアクセス

ワークスペース外の読み取りが必要なとき:

```text
/sandbox-add-read-dir C:\Users\you\shared-templates
```

書き込み先を増やす代わりに `danger-full-access` を選ばないでください。`writable_roots` や `--add-dir` で **必要最小限のパスだけ** 開放します。

## Windows 固有のサンドボックス

Windows ネイティブ実行時は、Linux/macOS とは別のサンドボックス実装が使われます。

```toml
[windows]
sandbox = "elevated"        # 推奨: 低権限ユーザー + ファイアウォール + ACL
# sandbox = "unelevated"    # フォールバック: 制限トークン + ACL
```

| モード | 特徴 |
|--------|------|
| `elevated` | 最も強い境界。管理者承認付きセットアップが必要 |
| `unelevated` | 管理者権限不要。ネットワーク隔離は弱い |

エラー `1385` が出た場合、サンドボックスユーザーのログオン権限がポリシーで拒否されています。IT 管理者に相談するか、一時的に `unelevated` で継続します。

診断ログ: `%USERPROFILE%\.codex\.sandbox\sandbox.log`

## 作業カテゴリ別の設計

### 許可してよい作業（workspace-write + on-request）

- ユニットテストの実行
- lint / formatter の実行
- 小さなソース修正（1〜3ファイル）
- ドキュメント整備
- AGENTS.md の更新

### 慎重に扱う作業（承認必須・事前に Git コミット）

- 依存関係の追加・更新（`npm install`, `cargo add` 等）
- DB マイグレーション
- ネットワークを伴うコマンド
- release ビルド
- CI/CD 設定の変更

### 禁止または明示承認のみ

- `.env` / 秘密鍵の表示・コミット
- ワークスペース外への書き込み
- 本番 DB への接続
- `git push --force`
- sandbox / approval 設定の無効化
- `rm -rf` 相当の破壊的操作

AGENTS.md にも同じ分類を書いておくと、Codex 側の判断が安定します。

## 非対話モードでの権限

`codex exec` はデフォルト **read-only** です。CI パイプラインでは **最小権限** を明示します。

```powershell
# 調査のみ（デフォルト相当）
codex exec "summarize failing tests"

# 編集を許可（隔離ランナー上）
codex exec --sandbox workspace-write --ask-for-approval never "fix the type error in src/main.rs"

# ユーザー設定を無視した制御下実行
codex exec --ignore-user-config --sandbox read-only "audit dependencies"
```

`--ignore-user-config` は、開発者の `~/.codex/config.toml` が緩い場合でも、CI 側で強制的に read-only にできる点で有用です。

## Approval は「不信」ではなく「責任の分割」

Approval プロンプトは、Codex を疑うための仕組みではありません。

- 人間が **責任を持てる単位** に作業を分ける
- 境界を越える操作（ネットワーク、外部書き込み）に **一時停止** を挟む
- チーム開発では **誰が何を許可したか** を意識させる

「全部自動」より「境界内は自動、境界外は確認」の方が、長期運用では速くて安全です。

## この章の完了条件

- [ ] 3 つの sandbox モードと 3 つの approval ポリシーを説明できる
- [ ] 日常開発・調査・CI 向けの組み合わせを選べる
- [ ] `config.toml` と `/permissions` で設定を固定・切り替えできる
- [ ] Windows の `elevated` / `unelevated` の違いを理解できる
- [ ] AGENTS.md の禁止事項と Sandbox 設定が矛盾していないことを確認できる

次章では、[zapabob/codex](https://github.com/zapabob/codex) を題材に、公式 Codex を基準とした **フォーク運用の読み方** を扱います。
