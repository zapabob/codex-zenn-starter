# グローバル AGENTS.md とリポジトリ AGENTS.md の違い

Codex には **2種類の AGENTS.md** があり、役割が異なります。

## リポジトリの AGENTS.md（本書の主題）

| 項目 | 内容 |
|------|------|
| 場所 | `your-repo/AGENTS.md` |
| 読み込み | `project_doc_fallback_filenames` 経由でプロジェクト文書として注入 |
| 目的 | **このリポジトリ** の test / lint / 禁止事項 / 完了条件 |
| 共有 | Git でチーム全員が同じルールを使う |
| サイズ目安 | 32 KiB 以内。実行可能なルールに絞る |

→ `examples/AGENTS.md.*.md` がこのタイプのテンプレートです。

## ユーザー全体の ~/.codex/AGENTS.md

| 項目 | 内容 |
|------|------|
| 場所 | `~/.codex/AGENTS.md`（Windows: `%USERPROFILE%\.codex\AGENTS.md`） |
| 読み込み | Codex ホームディレクトリのユーザー設定として適用 |
| 目的 | **全プロジェクト共通** の文体、スキル参照、MCP 利用方針、個人ワークフロー |
| 共有 | コミットしない。マシンごとの個人設定 |
| サイズ | 大きくなりがち（スキル一覧、ツール定義など） |

個人の `~/.codex/AGENTS.md` には、次のような内容が入ることがあります。

- 口調・文体のルール
- 利用可能スキルのパスと発火条件
- MCP / プラグイン利用時の優先順位
- ネットワーク・ファイルシステムの許可ドメイン（Codex 製品側設定の説明）

これらは **リポジトリの開発ルール** ではなく、**エージェントの振る舞い全般** を定義します。

## 使い分け

```
~/.codex/AGENTS.md     … 個人の Codex 全体設定（文体、スキル、MCP方針）
~/.codex/config.toml   … MCP サーバー定義、モデル、デフォルト sandbox

your-repo/AGENTS.md    … この repo 専用の test/lint/禁止事項
your-repo/.codex/config.toml … この repo 専用の sandbox / MCP 上書き
```

## よくある失敗

| 失敗 | 問題 | 対処 |
|------|------|------|
| 個人 AGENTS.md を repo にコミット | 1600行超の個人設定がチームに強制される | リポジトリ用に `examples/` テンプレから作り直す |
| repo AGENTS.md に文体だけ書く | test コマンドがなく Codex が迷う | Commands / Done を必ず書く |
| 両方に矛盾するルール | sandbox は config、禁止事項は AGENTS、で食い違い | 役割分担を固定する |

## 本リポジトリ（codex-zenn-starter）の AGENTS.md

Zenn 原稿リポジトリ自体にも AGENTS.md を置く場合は、`examples/AGENTS.md.node-nextjs.md` をベースに、次を追記します。

- `npm run preview` で Zenn プレビュー
- `published: true` 変更は意図的な公開のみ
- `_docs/` は実装ログ、`books/` / `articles/` が公開対象
