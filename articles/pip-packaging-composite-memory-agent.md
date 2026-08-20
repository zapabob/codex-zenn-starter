---
title: "自作AIエージェント記憶基盤をpipパッケージ化して再利用する設計ガイド"
emoji: "📦"
type: "tech"
topics: ["python", "pip", "uv", "aiagent", "architecture"]
published: true
---

## はじめに：スクリプト散乱からの脱却

AIエージェントの記憶基盤（Ebbinghaus忘却曲線、Obsidian Wiki連携、Semantic Graphなど）を構築していくと、最初は1つのプロジェクト内で直書きスクリプトとして運用しがちです。

しかし、エージェントが増えたり別プロジェクトで同じ記憶メカニズムを使い回したくなった際、**「ファイルをコピー＆ペーストして使い回す」状態はコードの二重管理と崩壊を招きます**。

この記事では、自作のAIエージェント記憶基盤を **Pythonパッケージ（pip化）** し、複数のプロジェクトやエージェントから一発で `import` または `pip install` して使い回せるようにする設計・パッケージ化ガイドを解説します。依存関係とビルドには現代の標準ツール **`uv`** を使用します。

---

## 1. ディレクトリ構成（`src` レイアウト推奨）

パッケージ化する際は、ルート直下にモジュールを置くのではなく、**`src/` ディレクトリ配下にパッケージを配置するレイアウト（src-layout）** を推奨します。テスト時に誤ってローカルディレクトリをインポートしてしまう事故を防ぐことができます。

```text
hermes-composite-memory/
├── pyproject.toml               # パッケージ定義・依存関係
├── README.md
├── LICENSE
├── src/
│   └── composite_memory/        # import composite_memory で呼び出すパッケージ本体
│       ├── __init__.py
│       ├── core.py              # 記憶基盤の統合インターフェース
│       ├── ebbinghaus.py        # 忘却曲線・記憶減衰ロジック
│       ├── obsidian.py          # Obsidian Markdown Wiki連携
│       ├── graph.py             # Semantic Graphノード検索
│       └── cli.py               # オプショナル：メンテナンス用CLI
└── tests/
    └── test_ebbinghaus.py
```

---

## 2. `pyproject.toml` によるモダンな構成定義

以前は `setup.py` が主流でしたが、現在は **`pyproject.toml`** による一元管理が標準です。ビルドバックエンドには高速かつシンプルな **Hatchling** を使用します。

```toml
[build-system]
requires = ["hatchling"]
build-backend = "hatchling.build"

[project]
name = "hermes-composite-memory"
version = "0.1.0"
description = "Ebbinghaus and Graph based Composite Memory Architecture for AI Agents"
readme = "README.md"
requires-python = ">=3.10"
authors = [
    { name = "Hakua", email = "your-email@example.com" }
]
license = { text = "MIT" }

# 記憶基盤が必要とする依存ライブラリ
dependencies = [
    "pydantic>=2.0.0",
    "networkx>=3.0",
    "gitpython>=3.1.0",
    "numpy>=1.24.0",
]

# ターミナルから記憶メンテナンスコマンドを動かしたい場合のCLIエントリーポイント設定
[project.scripts]
memory-cli = "composite_memory.cli:main"

[tool.hatch.build.targets.wheel]
packages = ["src/composite_memory"]
```

---

## 3. `uv` を活用したローカル検証と開発モード

依存関係管理・パッケージ操作には **`uv`** を使用します。

### ① ローカル開発モード（Editable Install）
開発中はコードを書き換えるたびにインストールし直すのは手間です。`-e`（Editable）オプションを付けてインストールすると、`src/` 配下のソースコードの変更が即座に反映されます。

```bash
# uv 環境で開発インストール
uv pip install -e .
```

### ② インポートと動作の確認

```python
from composite_memory import CompositeMemoryEngine

# エージェントの記憶基盤を初期化
memory = CompositeMemoryEngine(storage_path="./vault")
memory.recall(cue="検索クエリ")
```

---

## 4. GitHubから直接 `pip install` する運用（PyPI非公開でもOK）

「PyPIに一般公開するほどではない」「プライベートリポジトリで管理したい」という場合でも、`pyproject.toml` さえ用意しておけば、**GitHubのURLを指定して直接インストール**できます。

### コマンド1つで他プロジェクトに導入

```bash
# 通常のpipの場合
pip install git+https://github.com/zapabob/hermes-composite-memory.git

# uv を使うプロジェクトの場合
uv pip install git+https://github.com/zapabob/hermes-composite-memory.git
```

`pyproject.toml` や `requirements.txt` に以下のように記述することも可能です。

```toml
# 別のエージェントプロジェクトの pyproject.toml 内
[project]
dependencies = [
    "hermes-composite-memory @ git+https://github.com/zapabob/hermes-composite-memory.git@main",
]
```

これだけで、別プロジェクトで開発しているAIエージェントから、自作の複合記憶システムを共通ライブラリとして使い回せるようになります。

---

## 5. PyPIへ公式リリースする場合の流れ (`uv build` & `uv publish`)

世界中にパブリッシュしたい、または `pip install hermes-composite-memory` でインストールできるようにしたい場合は、`uv` のビルド・パブリッシュ機能を使用します。

```bash
# 1. パッケージのビルド (dist/ 配下に wheel ファイルが作られます)
uv build

# 2. PyPIへアップロード (APIトークンが必要です)
uv publish --token pypi-xxxxxxxxxxxx
```

---

## 6. パッケージ化によって得られる3つのメリット

1. **記憶ロジックの一元化**
   記憶減衰（Ebbinghaus）の計算式やグラフ検索のバグ修正を行った際、1箇所のリポジトリを更新して `pip install --upgrade` するだけで、すべてのAIエージェントに反映されます。
2. **エージェント本体コードの圧倒的クリーン化**
   エージェントのメイン処理に数百行の記憶管理ロジックが同居せず、`memory.remember()` や `memory.recall()` の数行に集約されます。
3. **テスタビリティの向上**
   `tests/` ディレクトリ内で記憶ロジック単体の単体テスト（pytestなど）が書きやすくなり、記憶システムの堅牢性が担保できます。

---

## おわりに

AIエージェントのアーキテクチャが高度化するにつれ、「エージェントのロジック」と「記憶・知能の基盤」を切り離してパッケージ化するアプローチは非常に強力です。

ぜひ自作の記憶エンジンを `pyproject.toml` ＋ `uv` でパッケージ化し、洗練されたエージェント開発環境を構築してみてください！
