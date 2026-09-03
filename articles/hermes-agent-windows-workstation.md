---
title: "「IDEにAIを足す」のをやめた。AIエージェントを中心にWindows環境を再構築した話"
emoji: "🖥️"
type: "tech"
topics: ["llm", "windows", "python", "oss", "aiagent"]
published: true
---

GitHub: https://github.com/zapabob/hermes-agent-windows

:::message
**【3行でわかるこの記事】**
1. 「IDEの横にAIチャットを生やす」アプローチに限界を感じ、**AIエージェントを作業の中心に据えてOS（Windows）環境そのものを組み直した**。
2. Git・Webブラウザ・ローカルLLM・長期複合記憶（忘却曲線）・そして**ClamAV/YARAによるセキュリティセンター**まで同一画面に統合。
3. デモで数分動くオモチャではなく、**24時間常駐して生き続けるエージェント**のためのWindows AIワークステーションをOSSとして開発中。
:::

---

## はじめに：「AI付きエディタ」への違和感

いま、開発者の多くがCursorやVS CodeのAI拡張を使っています。それらは間違いなく便利です。

しかし、自律型AIエージェントを本気で開発・運用しようとしたとき、強烈な違和感にぶつかりました。

> **「なぜ人間用のエディタの片隅に、無理やりAIを同居させているのか？」**
> **「AIエージェントが仕事をするなら、エージェントを中心にOS環境を再構築すべきではないか？」**

VS CodeにAIチャットを足すのではなく、**「AIエージェントの作業場（ワークステーション）の中に、IDE、ブラウザ、端末、ローカル推論、セキュリティを配置する」**。

この発想から生まれたのが、Nous ResearchのOSS「Hermes Agent」をベースにしたWindows-firstダウンストリーム、**Hermes Agent Windows Workstation Edition** です。

---

## まず画面を見てほしい

これが、現在開発しているワークステーションの実際の画面です。

![Security Center・X・YouTube・Gitリポジトリツリーを同時に表示したワークステーション画面](/images/hermes-workstation-security-center.png)
*中央にSecurity Centerのスキャン結果（2,617ファイル検査 / 32検出・検疫）、右側にブラウザとGitリポジトリツリーを同時配置*

![Hermes Agentホーム画面。エージェントを中心にブラウザ・Git・セッション管理を統合](/images/hermes-workstation-home.png)
*エージェントホーム画面。左側にセッション・Telegram・Discord・CRON、中央に対話、右上にWebブラウザ、右端にGitツリー*

一見すると「少し変わったIDE」に見えるかもしれません。
しかし、その設計思想は根本から異なります。

---

## パラダイムシフト：「IDE＋AI」から「Agent中心のWorkstation」へ

一般的なIDE（VS CodeやJetBrains）の中心にあるループはこれです。

```text
Code → Build → Test → Debug （人間が主役）
```

しかし、自律型AIエージェントがこなす実際の業務ループは、はるかに広大です。

```text
Observe → Research → Reason → Code → Test → Review → Operate → Automate → Recover
```

コードを書くのは、エージェントの仕事のほんの1ステップに過ぎません。

- リポジトリの構造を読み解く
- Webを検索して最新ドキュメントを当たる
- シェルコマンドを実行して挙動を確かめる
- MCP（Model Context Protocol）ツールを呼び出す
- 過去の記憶（Memory）を検索して教訓を引っ張り出す
- 定期ジョブを実行し、コケたらプロセスを自己復旧する

これを別々のウィンドウや別アプリでやらせると、コンテキストの分断と権限の混乱が起きます。
だからこそ、**これらすべてを同一のワークスペース上に集約しました。**

---

## ここが既存の環境と決定的に違う5つの特徴

### 1. 「IDE並みのGit操作」をエージェントと人間が共有する

デスクトップ右側にはGitリポジトリツリーが常駐しています。

単なるファイルビューアではありません。
- Git CRUD操作
- commit / diff / review
- branch / worktree 操作
- リポジトリの状態監視

```text
Hermes Agent
      │
      ├── Repository Tree
      ├── Git CRUD / Operations
      ├── Diff & Human Review
      ├── Terminal / Shell
      └── Coding Agent Core
```

右側で人間が差分を見ながら、左側のHermes Agentと相談し、同じ画面上で即座にコミットを打つ。
「エディタを開いて、ターミナルを開いて、Gitクライアントを開いて…」という往復はここにはありません。

### 2. ブラウザも「別アプリ」ではなくワークスペースの1ペイン

WebブラウザもOSの別ウィンドウではなく、エージェントと同じワークスペース内のペインとして統合されています。

エージェント開発をしていると、
```text
Webで一次情報調査 → リポジトリ確認 → 実装 → テスト → エラーを再検索
```
というループが何百回も発生します。
ブラウザが同じ画面にあることで、エージェント自身のブラウザ自動操作（Browser Automation）と人間の視線が完全に同期します。

### 3. なぜ「Security Center」が同居しているのか？（ここが一番の狂気）

このワークステーションの最大の特徴が、画面中央に鎮座する **Security Center** です。

| 監視・防御レイヤー | 使用エンジン・技術 |
| :--- | :--- |
| **ウイルス・マルウェア検知** | ClamAV / YARA / Windows Defender |
| **ファイル完全性検証** | Hash Reputation |
| **隔離・遮断** | Quarantine 管理 |
| **実行ログ・履歴** | Scan & Execution Audit History |

「なぜAIエージェントの画面にアンチウイルスや検疫画面があるのか？」と思うかもしれません。

理由は明快です。
**AIエージェントに強い権限（ローカルファイル書き換え、Shell実行、Git操作、MCP外部接続）を渡すなら、その実行環境をリアルタイムで観測・防御できなければ、怖くて24時間常駐などさせられないからです。**

おもちゃのデモならセキュリティは無視できます。しかし、本気で自律作業を任せるなら「防御と観測」はワークステーションの第一級機能でなければなりません。

### 4. 外部Go Watchdogによる「24時間死なない」常駐アーキテクチャ

AIエージェントをローカルで動かしたことがある人なら、誰もが経験したはずです。
「朝起きたらプロセスが落ちていた」「メモリリークで固まっていた」。

Hermes Agent Windowsでは、`scripts/windows/watchdog-go` に外部Go言語製Watchdogを配置しています。
ここで徹底している鉄則が、**「再起動の権限（Restart Authority）を唯一化すること」** です。

```text
       ┌──────────────────────┐
       │   Go Watchdog (唯一)  │ ── 死活監視 & 自動リカバリ
       └──────────┬───────────┘
                  │
  ┌───────────────┼───────────────┐
  ▼               ▼               ▼
Hermes Backend   Local Embedding   Desktop UI
```

Pythonプロセス、Electron、PowerShell、Goがそれぞれ勝手に自己再起動を試みると、カスケード障害を起こしてシステムが破綻します。
「外部のGo Watchdogだけが唯一の再起動権限を持つ」というシングル・オーソリティ設計により、24時間365日の連続稼働を可能にしています。

### 5. 「チャット履歴」を捨て、「忘却と意味グラフ」を持つ記憶モデル

「過去の会話履歴をプロンプトに全部突っ込む」のは、記憶（Memory）ではありません。ただのログ垂れ流しです。

本環境では、独自の **Semantic Graph（意味グラフ）** と **Ebbinghaus Cognitive Memory（忘却曲線モデル）** を実装しています。

```text
Conversation History ≠ Long-term Memory ≠ Semantic Retrieval
```

- **検索可能な記憶**: ベクトル検索 ＋ 知識グラフ ＋ 全文検索のハイブリッド
- **関係性リンク**: ノード間で意味が繋がるグラフ構造
- **忘却のメカニズム**: エビングハウスの忘却曲線に基づき、使われない記憶の重要度（Salience）が減衰
- **再活性化**: リハーサル（想起）されることで強固に定着する長期記憶

人間が「すべての出来事を丸暗記していないが、大事な教訓は覚えている」のと同じ構造をエージェントに与えています。

:::message
※ 複合記憶アーキテクチャの詳細は別記事「[RAGと何が違う？AIエージェントに同一性と忘却を与える複合記憶アーキテクチャ](/articles/composite-memory-vs-rag)」にまとめています。
:::

### 6. コーディングだけじゃない：Voice、Unity、VRChatへの受肉

このワークステーションは、黒い画面でコードを書くだけの環境ではありません。

```text
Hermes Agent
     │
     ├── Text / CLI
     ├── Voice (VOICEVOX / local TTS / 彩色TTS)
     ├── Unity
     └── VRChat (自律行動 / AITuber連携)
```

画面の中でテキストを吐くだけのAIではなく、音声で喋り、仮想空間（VRChatやUnity）へインタラクションを送る「受肉したエージェント」へ直結しています。Windowsネイティブだからこそ、デスクトップ上のあらゆるクリエイティブアプリやメディアパイプラインとシームレスに繋がります。

---

## 泥臭い裏側：Upstreamの猛スピードにどう追いつくか？

Nous ResearchのHermes Agent本体は、凄まじいスピードで開発が進んでいます。
単純に `git merge upstream/main` を繰り返すだけでは、Windowsダウンストリームの独自拡張は一瞬でコンフリクトして壊れます。

そこで、upstreamのコミットを1つずつ精査する「セマンティック統合」パイプラインを運用しています。

| 分類 | 処理方針 |
| :--- | :--- |
| **ADOPT** | そのまま取り込み |
| **COMPOSE** | Windowsの権限モデル・アーキテクチャと結合して統合 |
| **DEFER** | プラットフォーム互換性の観点から保留 |
| **KEEP_DOWNSTREAM** | ダウンストリーム固有の実装を維持 |

直近の統合キャンペーンの実績値です。

```text
Upstream 差分コミット : 1,049 件
├─ ADOPT (採用)        :   325 件
├─ COMPOSE (再構成)    :   723 件
└─ DEFER (保留)        :     1 件
ファイル衝突交差数     :   414 ファイル
```

1,000件以上のアップストリームコミットに対し、7割以上を「単なるコピペではなくWindowsの権限モデルに合わせて再合成（COMPOSE）」しながら追従しています。
「Windows-first」を謳う裏には、この泥臭いマージ戦略があります。


---

## 驚いたこと：まだReleaseを出していないのにClone数が…

先日、GitHubのTrafficアナリティクスを見て驚きました。

直近14日間の数字です。

| 指標 | 数値 |
| :--- | :--- |
| **Git clones** | **9,808** 回 |
| **Unique cloners** | **237** 人 |
| **Page views** | 227 pv |
| **Unique visitors** | 80 人 |

CIやボットによるクローンも含まれるため、237人全員が手動実行したとは言えません。
しかし、**インストーラーや正式Releaseバイナリすらまだ出していない、純粋なソースコード段階のリポジトリ**としては異様な数字でした。

……それなのに、**GitHubのStarはまだ「6」でした（笑）。**

「あ、これ何を作っているのか全然外に伝わってないな」と痛感しました。
コードだけゴリゴリ書いて満足していましたが、何を目指しているのかをちゃんと言語化しなければ届かない。それがこの記事を書いた理由です。

---

## 「OS上でAIを使う」から「AIがOSを作業場にする」未来へ

全体の構成を俯瞰すると、このようになっています。

```text
Hermes Agent Windows Workstation
│
├── Agent Runtime   : セッション / MCP / ツール / CRON自律実行
├── Engineering     : Gitツリー / CRUD / Diff / Review / Terminal
├── Browser         : 統合型Webワークスペース（調査・自動化）
├── Security        : ClamAV / YARA / Windows Defender / 検疫
├── Local AI        : llama.cpp / GGUF / Local Embeddings / 複合記憶
├── Media / Real    : Voice (VOICEVOX/TTS) / Unity / VRChat連携
└── Operations      : 外部Go Watchdogによる24時間常駐・自己修復
```

私たちはこれまで、「人間がOSを操作し、その中の1アプリとしてAIを使う」のが当たり前だと思っていました。

でも、自律型エージェントが本当に実用段階に入るなら、主客は逆転します。

> **「AIエージェントを中心に据え、その手足としてOSの全リソースを再配分する」**

このパラダイムで作るデスクトップ環境は、想像以上にエキサイティングです。

---

## リポジトリはこちら（OSS / MIT License）

開発はすべてオープンソース（MIT License）で進めています。

👉 **GitHub: [zapabob/hermes-agent-windows](https://github.com/zapabob/hermes-agent-windows)**

- WindowsでローカルAIや自律エージェントを本気で動かしたい方
- 「AI付きエディタ」の次の世界を一緒に作りたい方
- 単純に「この画面、なんか面白そう」と思ってくれた方

ぜひリポジトリを覗いてみてください。**StarやIssue、PR、大歓迎です！**

「AIを中心にOS環境を組み直すと何が起きるのか」。
ぜひ一緒にこの実験を面白がってもらえたら嬉しいです。
