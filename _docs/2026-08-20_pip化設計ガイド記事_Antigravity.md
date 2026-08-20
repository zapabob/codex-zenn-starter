# 2026-08-20 pip化設計ガイド記事作成 Antigravity

## 概要
自作のAIエージェント記憶基盤（Ebbinghaus, Obsidian, Graph等）をPythonパッケージ（pip）化し、別プロジェクトから再利用・運用するための設計ガイド記事をZenn形式で作成。

## 作成ファイル
- `articles/pip-packaging-composite-memory-agent.md`
  - Title: `自作AIエージェント記憶基盤をpipパッケージ化して再利用する設計ガイド`
  - Emoji: 📦
  - Type: tech
  - Topics: `python`, `pip`, `uv`, `aiagent`, `architecture`
  - Published: true

## 記事の主な構成・ポイント
1. `src` レイアウトによるモジュール設計
2. `pyproject.toml` による依存関係・Hatchlingビルド定義
3. `uv` を利用したローカル開発モード (`uv pip install -e .`)
4. PyPI非公開でも可能な `git+https://...` を使ったGitHub直接インストール運用
5. `uv build` および `uv publish` によるPyPI公開手順
6. パッケージ化によるメリット（ロジック一元化、コードクリーン化、単体テスト性向）
