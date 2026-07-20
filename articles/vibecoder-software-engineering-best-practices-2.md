---
title: "VibeCoderが陥る罠：可読性・環境変数・分割設計とAGENTS.md活用"
emoji: "🧹"
type: "tech"
topics: ["vibecoding", "cleancode", "python", "agentsmd", "refactoring"]
published: true
---

前回の記事（[いまさら聞けないVibeCoder向け：git・GitHub・PR・CodeRabbit入門](/articles/vibecoder-git-github-pr-coderabbit)）では、gitとGitHubの基礎からチーム作業の流れまでを整理しました。

今回は「コードが動いた」その先の話です。

AIは素早くコードを生成しますが、**AIが生成したコードをAIが読み直すとき**、人間と同じ困りごとを抱えます。意味不明な数値、どこから読めばいいか分からない1ファイル、埋め込まれた認証情報——これらはAIの精度も下げます。

この記事は、VibeCoderがよく踏む落とし穴と、AGENTS.mdおよびSOPによる対策をまとめた**ソフトウェア工学ベストプラクティス第2弾**です。

---

## 1. マジックナンバーをなくす

### 何が問題か

次のコードを見てください。

```python
if retry_count > 3:
    time.sleep(60)
```

`3`と`60`がどこから来た数字なのか、コードだけでは分かりません。これをマジックナンバーと呼びます。

AIにこのコードを渡して「リトライ間隔を変えて」と頼むと、AIは`60`を変えるべきか、他の`60`も変えるべきか、確信を持てません。同じ数値が複数箇所に散らばっていれば、変更漏れが起きます。

### 対策：定数として名前をつける

```python
MAX_RETRY_COUNT = 3
RETRY_INTERVAL_SECONDS = 60

if retry_count > MAX_RETRY_COUNT:
    time.sleep(RETRY_INTERVAL_SECONDS)
```

名前がついた定数は、コードの意図を説明します。AIも「`MAX_RETRY_COUNT`を5に変えて」という指示を迷わず実行できます。

定数はファイルの先頭か、専用の`constants.py`にまとめるとさらに管理しやすくなります。

✅ **Vibe Coder アクション:** コード内の数値リテラルを検索し、意味が自明でないものに名前をつけます。

---

## 2. 環境変数をコードに埋め込まない

### 何が問題か

```python
client = openai.OpenAI(api_key="sk-proj-xxxxxxxxxxxxxxxx")
```

これを公開リポジトリにpushした瞬間、APIキーは世界中から見えます。git historyから削除しても、fork・clone・キャッシュに残ります。漏えいしたキーは即座に無効化が必要です。

AIがコードを生成するとき、テスト用にダミーキーを埋め込む場合があります。その後、実キーへ書き換えたままcommitするのが典型的な事故です。

### 対策：`.env`と`python-dotenv`を使う

```bash
# .env（gitignoreに追加する）
OPENAI_API_KEY=sk-proj-xxxxxxxxxxxxxxxx
```

```python
import os
from dotenv import load_dotenv

load_dotenv()
api_key = os.getenv("OPENAI_API_KEY")
if not api_key:
    raise ValueError("OPENAI_API_KEY が設定されていません")
client = openai.OpenAI(api_key=api_key)
```

`.env`は必ず`.gitignore`に追加します。

```text
# .gitignore
.env
.env.*
!.env.example
```

`.env.example`は値を空にしたテンプレートとして公開します。

```text
# .env.example
OPENAI_API_KEY=
DATABASE_URL=
```

:::message alert
`OPENAI_API_KEY`などの文字列がコードに含まれていないか、commitの前に `git diff --staged` で確認する習慣をつけてください。`git-secrets`（前回記事参照）による自動検知と組み合わせると安全です。
:::

✅ **Vibe Coder アクション:** プロジェクトに`.env.example`を作り、`.env`が`.gitignore`にあるか確認します。

---

## 3. スパゲッティコードを避ける

### 何が問題か

スパゲッティコードとは、処理の流れが複雑に絡み合い、どこから何が始まり、どこで何が終わるか追いにくいコードです。

AIは長い関数を渡されると、変更の影響範囲を把握しにくくなります。100行を超える関数、グローバル変数を多用した処理、条件分岐が7段ネストした`if`文——これらはAIの生成精度を落とします。

### 対策：関数は1つのことだけを行う

```python
# ❌ 1つの関数が取得・加工・保存・通知をすべて行う
def process_user_data(user_id):
    # APIからデータ取得
    # データ加工
    # DB保存
    # Slack通知
    ...

# ✅ 役割を分けた関数
def fetch_user_data(user_id: str) -> dict:
    ...

def normalize_user_data(raw: dict) -> dict:
    ...

def save_user(data: dict) -> None:
    ...

def notify_slack(message: str) -> None:
    ...
```

1つの関数が1つのことをすれば、「`fetch_user_data`だけ変えて」という指示がAIに正確に届きます。

関数の長さの目安は**20〜40行**です。超えてきたら、分割を検討します。

✅ **Vibe Coder アクション:** 一番長い関数を1つ選び、処理を3つの小さな関数に分けます。

---

## 4. 1ファイル1000行を超えたら分割する

### 何が問題か

`main.py`に全機能を詰め込んで3000行になったファイルは、次の問題を引き起こします。

* どこに何があるか分からない
* AIが全文を読みきれず、途中だけ見て変更する
* 複数人・複数AIが同じファイルを変更すると必ずコンフリクトする

### 対策：機能ごとにファイルを分ける

1000行を超えてきたら、関心ごとの単位でファイルを分割します。

```text
# ❌ 一枚岩の構造
my_app/
└── main.py   # 3000行

# ✅ 関心ごとに分割した構造
my_app/
├── main.py          # エントリーポイントのみ（〜50行）
├── config.py        # 設定・定数
├── models/
│   ├── user.py
│   └── session.py
├── services/
│   ├── auth.py
│   └── notification.py
└── utils/
    ├── logging.py
    └── validators.py
```

分割の基準は「このファイルの1行目を読めば、このファイルが何をするか分かるか」です。

### モノレポ構成を避ける

モノレポとは、複数の独立したアプリやサービスを1つのリポジトリに詰め込む構成です。小規模プロジェクトでは管理が複雑になりやすく、AIが「このファイルはどのサービスの一部か」を判断しにくくなります。

最初は**1リポジトリ = 1サービス**の原則を守ります。

```text
# ❌ 小規模なのにモノレポ
my-workspace/
├── frontend/
├── backend/
├── ml-pipeline/
└── infra/

# ✅ 最初はシンプルに
my-backend/          # リポジトリ1つ
my-frontend/         # リポジトリ1つ
```

プロジェクトが成長してモノレポが必要になったとき、そのとき移行を検討します。

✅ **Vibe Coder アクション:** 一番大きいファイルの行数を `wc -l <ファイル名>` で確認します。800行を超えていれば分割候補です。

---

## 5. docstringで意図を残す

### 何が問題か

次の関数は何をしますか？

```python
def calc(x, y, mode):
    if mode == 1:
        return x + y
    elif mode == 2:
        return x * y
    else:
        return x - y
```

関数名`calc`、引数名`x y mode`、戻り値の型——すべてが不明です。AIにこの関数の修正を頼むと、`mode`の意味から推測させることになります。

### 対策：Google スタイルのdocstringを書く

```python
def calculate(value_a: float, value_b: float, mode: str) -> float:
    """2つの数値を指定した演算モードで計算する。

    Args:
        value_a: 演算対象の第1の数値。
        value_b: 演算対象の第2の数値。
        mode: 演算の種類。"add"（加算）、"mul"（乗算）、"sub"（減算）のいずれか。

    Returns:
        演算結果の数値。

    Raises:
        ValueError: modeが未定義の値の場合。

    Example:
        >>> calculate(3.0, 2.0, "add")
        5.0
    """
    if mode == "add":
        return value_a + value_b
    elif mode == "mul":
        return value_a * value_b
    elif mode == "sub":
        return value_a - value_b
    else:
        raise ValueError(f"未定義のmode: {mode}")
```

docstringの効果は次のとおりです。

* AIが関数の意図を正確に把握して修正できる
* IDEの補完に表示される
* `help(calculate)`で確認できる
* `pdoc`などでAPIドキュメントを自動生成できる

型ヒントとdocstringはセットで書きます。型ヒントは「何の型か」、docstringは「なぜそうするか」を伝えます。

✅ **Vibe Coder アクション:** 一番よく使う関数にdocstringを1つ追加します。AIに「Google スタイルでdocstringを書いて」と頼むのが早道です。

---

## 6. 可読性の高いコードはAIも迷わない

「読みやすいコードはAIに優しい」という視点は、Vibe Codingで特に重要です。

AIは変更依頼を受けると、ファイル全体またはその一部をコンテキストとして読みます。そのとき次のコードはどう見えるでしょうか。

```python
# ❌ AIが意図を読み取りにくいコード
def p(d, f=True):
    r = []
    for i in d:
        if f:
            if i["s"] == 1:
                r.append(i)
        else:
            r.append(i)
    return r
```

```python
# ✅ AIが意図を読み取りやすいコード
def filter_active_users(users: list[dict], active_only: bool = True) -> list[dict]:
    """ユーザーリストから条件に合うユーザーを返す。

    Args:
        users: ユーザー辞書のリスト。各要素は {"status": int, ...} を持つ。
        active_only: Trueのとき、status=1（アクティブ）のユーザーのみを返す。

    Returns:
        条件に合うユーザーのリスト。
    """
    if not active_only:
        return users
    return [user for user in users if user["status"] == 1]
```

後者は、AIが「`active_only`の条件を変えて」という依頼を安全に実行できます。前者は、`f`が何かを推測させることになります。

**可読性はAIへの指示精度に直結します。**

---

## 7. AGENTS.mdとSOPで対策を仕組みにする

個人の注意だけでは、ルールは徐々に崩れます。AIへの依頼ごとにプロンプトで説明するのも非効率です。

対策を仕組みにする場所が、**AGENTS.md**と**SOP**です。

### AGENTS.mdに書くべきこと

`AGENTS.md`はリポジトリのルートに置くファイルで、AIエージェントへの行動規範を記述します。Codex、Claude、Geminiなど多くのAIエージェントがこのファイルを読み、指示に従います。

次の項目を記述することで、AIが自動的にベストプラクティスに従ったコードを生成するようになります。

```markdown
## コーディング規約

- マジックナンバー禁止：数値リテラルは定数として定義し、意味のある名前をつける
- 環境変数はコードに直接書かない：`os.getenv()` と `.env` を使う
- 関数は1つの責任のみを持つ（単一責任の原則）
- 1ファイルは原則1000行以内。超えた場合はモジュール分割を提案する
- 型ヒントとGoogle スタイルのdocstringを必ず書く
- `print`禁止。`logging`モジュールを使う
- モノレポは避ける：1リポジトリ1サービスを原則とする

## 禁止事項

- APIキー、パスワード、トークンをコードに直接記述すること
- `except: pass` による例外の握りつぶし
- グローバル変数への代入（定数は除く）
```

このAGENTS.mdがあれば、AIへ毎回「環境変数をハードコードしないで」と言わなくて済みます。

### SOPで手順を標準化する

SOP（Standard Operating Procedure）は、繰り返す作業の手順書です。

たとえば「新しいスクリプトを作るときの手順」をSOPにまとめます。

```markdown
# SOP: 新規Pythonスクリプト作成手順

1. ブランチを作る：`git checkout -b feature/<機能名>`
2. Linearにチケットを作り、ブランチ名にID（例: `LIN-xxx`）を含める
3. ファイルを作る
   - モジュール先頭に `"""モジュールの目的を1行で書く。"""` を追加
   - 定数はファイル上部にまとめる
   - 環境変数は `os.getenv()` で取得し、Noneチェックをする
4. 関数1つあたり40行を超えたら分割を検討する
5. commitする前に `git diff --staged` で差分を確認する
6. PRを作り、`@coderabbitai review` を実行する
```

手順がSOPにまとまっていれば、AIへ「SOPに従って新しいスクリプトを作って」と依頼するだけで、ベストプラクティスに沿った雛形を生成させられます。

:::message
このリポジトリでは `AGENTS.md` がルートに、SOPが `_docs/sop/` 以下に配置されています。自分のプロジェクトにも同じ構成を作ると、AIとの作業が格段に整理されます。
:::

✅ **Vibe Coder アクション:** `AGENTS.md`の `## コーディング規約` セクションに、今日から守りたいルールを1つ追加します。

---

## まとめ：AIに迷わせないコードを書く

| 問題 | 対策 |
|------|------|
| マジックナンバー | 定数に名前をつけ、`constants.py`にまとめる |
| 環境変数のハードコード | `.env` + `os.getenv()` + `.gitignore` |
| スパゲッティコード | 関数を1責任・20〜40行に分割する |
| 1ファイル肥大化 | 1000行を超えたらモジュール分割する |
| docstringなし | 型ヒント + Google スタイルdocstringを必ず書く |
| モノレポ | 1リポジトリ1サービスを守る |
| ルールが守られない | `AGENTS.md` と SOP に仕組みとして書く |

Vibe Codingの速度は魅力です。ただし、AIが生成したコードを次のAIが修正し、さらに次のAIがレビューするサイクルが回るとき、コードの可読性はシステム全体の精度に影響します。

AIに優しいコードは、人間にも優しいコードです。
