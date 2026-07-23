---
title: "zapabob/codexを読む"
free: true
---

# zapabob/codexを読む

この章では、公式 [openai/codex](https://github.com/openai/codex) だけでなく、[zapabob/codex](https://github.com/zapabob/codex) を題材に、**upstream-first フォーク** の読み方と運用の考え方を扱います。

zapabob/codex は公式 Codex の **置き換え** ではありません。公式を基準にしながら、拡張を分離し、検証可能な形でリリースする —— その実例として読みます。

## リポジトリの位置づけ

| 項目 | openai/codex | zapabob/codex |
|------|-------------|---------------|
| 役割 | 公式 Codex CLI / app-server / プラグイン基盤 | upstream-first フォーク + 実験的拡張 |
| ライセンス | Apache-2.0 | 同上（fork） |
| 最新リリース（現行ライン） | `rust-v0.144.5` 系 | `v3.4.0`（2026-07-17） |
| 同期基準 | — | 公式 `rust-v0.144.5` + upstream main |

zapabob/codex の `CHANGELOG.md` によると、現行 **v3.4.0** では次が行われています。

- 公式 Codex `rust-v0.144.5` および upstream main (`71448a29e...`) とのリベース同期
- 単一正本としてのルート `VERSION` ファイル採用および `version-metadata.json` による競合解消マッピング
- GPT-5.6 ファミリーモデル・API サポートの組み込み
- `codex-ollama` プロバイダによるローカル・オフラインモデル動作
- Git4D 空間コードワークフロー（WebXR / VR / AR セッション API ブリッジ）
- スレッドゴール API（Python SDK）および Goal Status UI の維持・統合

フォークは **常に upstream より遅れまたは進む** 状態になります。使う前に CHANGELOG と release tag を確認してください。

## upstream-first とは

upstream-first fork では、次の順序を守ります。

1. **公式 openai/codex の変更を取り込む**（merge / rebase / sync script）
2. **コンフリクトを overlay ルールで解決** する
3. **フォーク固有の拡張だけ** を `plugins/`、`extensions/`、`zapabob/` 等に置く
4. **テストと release evidence** を残してからタグを切る

zapabob/codex では `scripts/upstream_sync.py` が authoritative な sync / release-closeout ドライバとして README / CHANGELOG に明記されています。

```
公式 openai/codex (upstream)
        │
        ▼ upstream_sync.py / overlay merge
zapabob/codex (main)
        │
        ├── plugins/          … プラグインバンドル
        ├── .codex/agents/    … エージェント定義
        ├── extensions/       … 拡張機能
        └── zapabob/          … フォーク固有スクリプト
```

**upstream 本体のコードに拡張を直書きしない** —— これがフォークを壊さない最大のルールです。

## 読む順番

最初から `codex-rs/` の実装詳部に入ると迷います。次の順で読みます。

### 1. README と CHANGELOG

- [README.md](https://github.com/zapabob/codex/blob/main/README.md) — 現行の製品面（`codex`, `codex app`, `codex app-server`, plugins）
- [CHANGELOG.md](https://github.com/zapabob/codex/blob/main/CHANGELOG.md) — 同期した upstream バージョンとフォーク固有の Added/Changed

「今のフォークは公式のどの時点をベースにしているか」をここで把握します。

### 2. AGENTS.md

公式・フォーク共通で、Rust モノレポ向けの **AI 向け開発規約** が書かれています。

- クレート命名（`codex-` プレフィックス）
- サンドボックス環境変数 `CODEX_SANDBOX_*` を安易に変更しない
- `just write-config-schema` を config 変更後に実行
- Bazel / `just` ベースの検証コマンド

AGENTS.md は、Codex リポジトリ自身が「AI エージェント向け設計書」をどう書くかの実例でもあります（第3章参照）。

### 3. リポジトリレイアウト

主要ディレクトリ:

| パス | 内容 |
|------|------|
| `codex-rs/` | Rust 製 CLI / core / app-server |
| `codex-cli/` | TypeScript ラッパー |
| `plugins/` | プラグインバンドル（例: `zapabob-legacy-suite`） |
| `.agents/plugins/` | マーケットプレイス manifest |
| `.codex/agents/` | YAML エージェント定義（architect, code-reviewer 等） |
| `scripts/` | upstream sync、release、検証 |
| `zapabob/scripts/` | フォーク固有（`load-env.ps1`, completion sound 等） |
| `docs/` | 開発ログ・設計メモ（`_docs/` 系） |

公式 openai/codex に無い `.codex/agents/` や `plugins/zapabob-legacy-suite/` が **拡張の境界** です。

### 4. upstream との差分

GitHub compare で公式 main との差分を見ます。

```
https://github.com/zapabob/codex/compare/openai:main...zapabob:main
```

差分の読み方:

- **added** — フォーク固有の新規ファイル（拡張候補）
- **modified in codex-rs/** — upstream 取り込み時のコンフリクト解消箇所（要注意）
- **大量の behind** — 公式が進んでいるので、次の sync が必要

フォーク利用者は「ahead した独自機能」と「behind した公式修正」の両方を意識します。

### 5. ビルドと検証

Rust 本体のビルド（WSL2 または Linux 推奨）:

```bash
cd codex-rs
cargo build -p codex-cli
cargo test -p codex-core
```

Windows ネイティブ検証では、CHANGELOG に記載の **`v8` symlink 権限** など Windows 固有の前提があります。release 失敗が repo regression か環境問題かを切り分けます。

フォーク固有の環境読み込み:

```powershell
# zapabob/scripts/load-env.ps1
. .\zapabob\scripts\load-env.ps1
```

## フォーク固有の拡張を理解する

### プラグインとエージェント

v3.1.0 以降、レガシー GUI（`codex gui-x`）は非推奨となり、**公式プラグイン機構** が製品 baseline です。

- `.agents/plugins/marketplace.json` — リポジトリ内マーケットプレイス
- `plugins/zapabob-legacy-suite/.codex-plugin/plugin.json` — レガシー機能のプラグイン化
- `.codex/agents/*.yaml` — サブエージェント定義（architect, dependency-analyst 等）

拡張を追加するときは **新しいプラグイン** または **新しい agent YAML** として切り出し、core を fork しない方針が CHANGELOG と一致しています。

### Git4D / app-server

v3.2.0 で app-server に Git4D 関連メソッド（capabilities, session start/list/watch/unwatch）が追加されています。AR/VR 向け拡張は **app-server プロトコル** として切り出されており、CLI 本体とは別レイヤーです。

本書の Codex 安全運用の文脈では、「拡張は app-server / plugin 境界に置く」という **分離の見本** として参照します。日常の AGENTS.md / Sandbox 設計とは独立したトピックです。

## リリースチャネル

CHANGELOG より:

| チャネル | タグ例 | 用途 |
|---------|--------|------|
| stable | `v3.1.0-stable.0` | 追従重視・検証済み |
| mainline | `v3.2.0` | 最新 upstream 同期 + 拡張 |

フォークをインストールする場合、**どのチャネルのバイナリか** を README / Releases ページで確認してから使います。公式 npm / install.sh と混同しないでください。

## 公式 Codex とフォークの使い分け

| シーン | 推奨 |
|--------|------|
| 初めて Codex CLI を試す | 公式 `openai/codex` の install 手順 |
| 日常開発 | 公式リリース or ChatGPT 経由の公式 CLI |
| フォーク拡張の検証 | zapabob/codex の release tag |
| 拡張機能の開発 | zapabob/codex を clone し upstream sync ワークフローに従う |

フォークは **学習と拡張の題材** であり、チーム全員に配布する実行バイナリである必要はありません。

## 壊れにくい fork 運用チェックリスト

- [ ] 定期的に `scripts/upstream_sync.py`（または同等）で upstream を取り込む
- [ ] 拡張は `plugins/` / `zapabob/` に閉じる
- [ ] CHANGELOG に同期した upstream バージョンを記録する
- [ ] Windows / Linux 両方の検証ログを release に残す
- [ ] セキュリティアドバイザリを upstream 取り込み時に確認する
- [ ] AGENTS.md に「触ってはいけない upstream 領域」を書く

## この章の完了条件

読者が次を説明できる状態を目指します。

- [ ] openai/codex と zapabob/codex の関係（upstream-first fork）
- [ ] CHANGELOG から同期 baseline バージョンを読み取れる
- [ ] 拡張コードが置かれるディレクトリ（plugins, .codex/agents, zapabob）を指せる
- [ ] compare URL で upstream との ahead/behind を確認できる
- [ ] 公式 CLI とフォークの使い分けを判断できる

---

## 次の章へ

第5章は **読む** ところまでです。拡張の手順は第8章「フォークを壊さず拡張する」、全体のまとめは第9章で扱います。

MCP（第6章）と CI（第7章）も公開済みです。`examples/` のテンプレートをコピーして、自リポジトリに展開してください。
