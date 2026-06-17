---
title: "フォークを壊さず拡張する"
free: true
---

# フォークを壊さず拡張する

第5章では zapabob/codex を **読む** 視点で扱いました。

この章では、upstream-first フォークに **自分の拡張を足す** ときの手順と判断基準を扱います。公式 openai/codex のコアを書き換えず、拡張だけを分離して検証可能にする —— それがゴールです。

## 拡張を置く場所

フォークで触ってよい領域と、触らない領域を先に決めます。

| 領域 | 例 | 方針 |
|------|-----|------|
| upstream コア | `codex-rs/codex-core/` | **直書きしない**（バグ修正以外は upstream へ） |
| プラグイン | `plugins/` | 機能バンドル・MCP 連携の追加 |
| エージェント定義 | `.codex/agents/*.yaml` | サブエージェント・ワークフロー定義 |
| フォーク固有 | `zapabob/` | スクリプト、検証ログ、リリース補助 |
| 設定例 | `examples/`（本リポジトリ） | 読者向けテンプレ（フォーク本体とは別） |

**ルール:** 拡張は常に「差し替え可能な層」に置く。コアに if fork 分岐を増やさない。

## 最小の拡張フロー

新しい拡張を足すとき、次の順で進めます。

### 1. upstream を最新にする

```bash
git fetch upstream
python scripts/upstream_sync.py   # zapabob/codex の authoritative ドライバ
```

コンフリクトが出たら、**upstream 側の意図を優先** しつつ、overlay ルール（`scripts/upstream_overlay_merge.py` 等）に従って解消します。CHANGELOG に同期した upstream タグを必ず記録します。

### 2. 拡張用ブランチを切る

```bash
git checkout -b feat/my-plugin-name
```

1 PR = 1 拡張単位にします。AGENTS.md・Sandbox・テスト・CHANGELOG を同じ PR に含めます。

### 3. エージェント定義を足す（任意）

`.codex/agents/` に YAML を置き、Codex から呼び出すサブエージェントを定義できます（第6章参照）。

```yaml
# .codex/agents/example-reviewer.yaml（概念例）
name: example-reviewer
description: Review changes against AGENTS.md rules
```

公式のスキーマ・必須フィールドは upstream の `docs/` または既存の `.codex/agents/*.yaml` をコピーしてから編集してください。

### 4. プラグインを足す（任意）

プラグインは `plugins/<name>/` に閉じます。`plugin.json`（または upstream 準拠のマニフェスト）でエントリポイントを宣言し、marketplace 登録手順は upstream README に従います。

**やってはいけないこと:**

- `codex-rs/` 内にフォーク専用クレートを直置きする（後の sync で毎回壊れる）
- 個人の `~/.codex/config.toml` の内容をリポジトリにコミットする

### 5. AGENTS.md を更新する

フォーク向けテンプレは [examples/AGENTS.md.codex-fork.md](https://github.com/zapabob/codex-zenn-starter/blob/main/examples/AGENTS.md.codex-fork.md) をベースにします。

最低限書く項目:

```md
## Rules
- Extension code belongs in `plugins/`, `extensions/`, or `zapabob/` — not inline in upstream-owned core.
- Upstream sync: `python scripts/upstream_sync.py` before release tags.
```

Codex に「触ってはいけないディレクトリ」を AGENTS.md で明示すると、自動化時の事故が減ります。

### 6. 検証してからタグを切る

zapabob/codex では release evidence を重視します。

- [ ] 影響範囲の `cargo test -p <crate>`
- [ ] config 変更後は `just write-config-schema`
- [ ] Windows / Linux の少なくとも一方で手動スモーク（`codex --version`、`codex exec` の基本動作）
- [ ] CHANGELOG に Added / Changed と upstream baseline を記載

検証ログは `_docs/` や release ノートに残します（Zenn 原稿や公開 README には内部メモを混ぜない）。

## upstream 取り込み時のコンフリクト

よくあるパターン:

| 状況 | 対処 |
|------|------|
| 同じファイルを upstream と拡張の両方が変更 | 拡張側を `plugins/` へ移すリファクタを検討 |
| `Cargo.lock` / 依存バージョン | upstream に合わせ、拡張クレートだけ差分を残す |
| `AGENTS.md` | upstream の更新を取り込み、フォーク固有セクションだけ追記 |
| セキュリティアドバイザリ | upstream の修正を **先に** マージしてから拡張 PR を rebase |

「とりあえず ours/theirs で片方を捨てる」は、次の sync で再発します。overlay ルールに従うか、拡張の置き場所を見直します。

## 本リポジトリ（codex-zenn-starter）との関係

この Zenn 本リポジトリは **フォーク本体ではありません**。役割分担は次のとおりです。

| リポジトリ | 役割 |
|-----------|------|
| openai/codex | 公式 CLI・仕様の正本 |
| zapabob/codex | upstream-first フォークの実装例 |
| codex-zenn-starter | 本書の原稿 + `examples/` テンプレ + CI dogfooding |

第7章の PR レビュー workflow は、本書の Markdown 品質を Codex に検証させる **実験場** です。フォーク開発の CI とは secret や権限設計を共通化できますが、リポジトリを混同しないでください。

## Windows での拡張検証

公式・フォークとも Windows サポートは進んでいますが、フォーク固有のビルドスクリプトは Linux 前提のことがあります。

- 日常の執筆・レビュー: Windows 11 + 公式またはフォークの `codex` バイナリ
- CI: `ubuntu-latest` で Codex ジョブを回す（第7章）
- リリース前: README が要求する Windows 手順（`cargo build`、インストーラスクリプト等）を **実機で一度** 実行

「Windows-first」と言うなら、ログを残すことまでセットです。

## この章の完了条件

- [ ] 拡張コードを `plugins/` / `.codex/agents/` / `zapabob/` に閉じられる
- [ ] `scripts/upstream_sync.py` を取り込みフローの入口にできる
- [ ] コンフリクト時に「コア直書き」を避ける判断ができる
- [ ] AGENTS.md.codex-fork テンプレをフォークに適用できる
- [ ] 検証ログと CHANGELOG を release の一部として扱える

---

次章（第9章）では、第1章からここまでの内容を **一つの開発作法** としてまとめます。
