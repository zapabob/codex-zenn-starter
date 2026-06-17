---
title: "AGENTS.mdでリポジトリに規律を与える"
free: true
---

# AGENTS.mdでリポジトリに規律を与える

AGENTS.mdは、単なるプロンプト置き場ではありません。

Codexが迷わず作業するために、リポジトリの構造、実行方法、テスト、lint、完了条件、禁止事項を明示する設計ファイルです。

## AGENTS.mdに書くこと

最低限、次の項目を書きます。

- リポジトリの目的
- ディレクトリ構造
- セットアップ方法
- テスト方法
- lint / format方法
- 変更してよい場所
- 変更してはいけない場所
- secretsの扱い
- 完了条件

## 悪いAGENTS.mdの例

```md
# AGENTS.md

いい感じに実装してください。
テストも適当にお願いします。
```

これでは、Codexは判断基準を持てません。

## よいAGENTS.mdの最小例

```md
# AGENTS.md

## Project
This repository contains a Python CLI application.

## Commands
- Install: `uv sync`
- Test: `uv run pytest`
- Lint: `uv run ruff check .`
- Format: `uv run ruff format .`

## Rules
- Do not read or print secrets from `.env`.
- Do not modify files under `data/raw/`.
- Keep changes small and explain the test result.

## Done
- Tests pass.
- Lint passes.
- The final response includes changed files and verification commands.
```

## Codexにとって読みやすい指示

AGENTS.mdでは、抽象的な精神論より、実行可能なルールを優先します。

よい書き方は、次のようなものです。

- 何を実行するか
- どこを見るか
- 何を変更しないか
- どの状態で完了とするか

## この章で作るテンプレート

この本では、最終的に次のテンプレートを作ります。

- Python AI Agent用
- Rust CLI用
- Unity/C#用
- Next.js用
- zapabob/codex fork用
