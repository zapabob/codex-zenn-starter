---
title: "Hermes Agentを「Windows AIワークステーション」にした話"
emoji: "🖥️"
type: "tech"
topics: ["llm", "windows", "python", "oss", "aiagent"]
published: true
---

GitHub: https://github.com/zapabob/hermes-agent-windows

私は現在、Nous ResearchのOSSである **Hermes Agent** をベースにしたWindows-first downstream、**Hermes Agent Windows Workstation Edition** を開発しています。

最初は「Hermes AgentをWindowsで安定して動かしたい」というところから始まりました。

しかし、ローカルLLM、長期Memory、ブラウザ、Git、Security Center、Voice、VRChat/Unity、24時間稼働のWatchdogなどを追加していった結果、現在は単純な「Hermes AgentのWindows版」という説明では実態と合わなくなってきました。

いま作っているものを一言で表すなら、

> **WindowsそのものをAI Agentのワークステーションにする環境**

です。

---

## 現在の画面

中央ではHermes Agentとの会話やSecurity Centerを表示し、右側ではWebブラウザとGit repository treeを同時に開いています。

一見するとIDEに近いのですが、設計思想は少し違います。

一般的なIDEにAIを追加するのではなく、

> **AI Agentを中心に、その周囲へIDE、ブラウザ、セキュリティ、ローカル推論、Memory、Automationを統合する**

という方向で作っています。

---

## AI付きIDEではなく、Agent中心のWorkstation

VS CodeやJetBrains系IDEをかなり単純化すると、

```text
Code → Build → Test → Debug
```

が中心になります。

Hermes Agent Windowsでは、これをもう少し広く捉えています。

```text
Observe → Research → Reason → Code → Test → Review → Operate → Automate → Recover
```

コードを書くことも重要ですが、それはAI Agentが行う仕事の一部分です。

実際のAgentには、以下のような仕事も必要になります。

- Git repositoryを読む
- Webを調査する
- Shell commandを実行する
- MCP toolを呼ぶ
- Memoryを検索する
- 定期ジョブを動かす
- 外部サービスへ接続する
- 失敗したprocessを復旧する

そのため、Hermes Agent Windowsではこれらを**同じworkspace上**へ集めています。

---

## IDE相当のGit操作

現在DesktopにはGit repository treeを表示できます。単なるtree viewerではなく、以下の操作も統合しています。

- Git CRUD
- commit / diff / review
- branch / worktree
- repository状態確認

右側のrepository treeからファイルを確認しながら、左側ではHermes Agentと相談し、同じDesktop上で変更をcommitできます。

```text
Hermes Agent
      │
      ├── Repository tree
      ├── Git operations
      ├── Diff / Review
      ├── Terminal
      └── Coding Agent
```

重要なのは、Git UIだけ独立したIDEを作っているのではないことです。Hermes Agent自身がrepositoryを理解し、toolを使い、その結果を同じworkspaceで人間が確認できるようにしています。

---

## BrowserもWorkspaceの一部

Webブラウザも別アプリではなく、workspace paneとして扱えます。

AI Agentを実際に使っていると、

```text
Webで調査 → repository確認 → 実装 → テスト → 再調査
```

という往復が非常に多くなります。そこでBrowserもAgent workspaceの一部として扱うことにしました。Hermes Agent本体のbrowser automationとも組み合わせられます。

---

## Security CenterをAI Agentと同じ画面に置く

AI Agentへ強いtool authorityを与えるほど、Securityは重要になります。

Hermes Agent Windowsには独自の **Security Center** を追加しています。現在は以下を一つの画面から確認できます。

| 項目 | 詳細 |
|------|------|
| ウイルス検知 | ClamAV / YARA / Windows Defender |
| ファイル検証 | hash reputation |
| 検疫 | quarantine管理 |
| 履歴 | scan history |

Agentがshell、browser、Git、MCP、local filesystemへアクセスできる以上、

> 「Agentが何をできるか」だけではなく、「その実行環境をどう観測し、どう防御するか」もWorkstationの機能

だと考えています。

---

## Local LLMを第一級のruntimeとして扱う

Hermes Agent WindowsではCloud APIだけではなく、local inferenceも重視しています。

現在扱っているもの：

- llama.cpp / GGUF
- local embeddings
- provider fallback
- hot-swap / hot-standby

特にWindows AI workstationでは、Cloud LLM＋Local LLM＋Local Embeddingを同時利用できる構成が便利です。

```text
Generation      → 高速なCloud/Local provider
Memory Retrieval → Local embeddings
Fallback        → Local llama.cpp
```

モデルAPIが一時的に利用できなくても、Workstation全体まで停止しない設計を目指しています。

---

## Memoryも単なるChat Historyではない

Hermes Agent本体にもMemory infrastructureがありますが、Windows downstreamではさらに独自のMemory実装を追加しています。

代表的なのが、**Semantic Graph** と **Ebbinghaus Cognitive Memory** です。

```text
Conversation History ≠ Long-term Memory ≠ Semantic Retrieval
```

大量のチャットログをそのままpromptへ投入するのではなく、以下の仕組みをAgent側へ持たせています。

- **検索可能な記憶**（ベクトル＋グラフ＋全文検索の多経路）
- **関係性**（ノード間の意味リンク）
- **忘却**（Ebbinghaus忘却曲線によるSalience減衰）
- **再活性化**（リハーサルによる重要記憶の強化）

:::message
複合記憶モデルとRAGの違いについては、別記事「[RAGと何が違う？AIエージェントに同一性と忘却を与える複合記憶アーキテクチャ](/articles/composite-memory-vs-rag)」で詳しく解説しています。
:::

---

## Voice / VRChat / Unity

このWorkstationはCodingだけを対象にしていません。

```text
Hermes Agent
     │
     ├── Text
     ├── Voice（VOICEVOX / local TTS / Irodori TTS）
     ├── Unity
     └── VRChat（autonomy / AITuber integrations）
```

Agentが単なるCLI chatbotではなく、Windows上で動作する複数のアプリケーションや仮想空間と接続できるようにしています。

---

## 24時間動かすためのGo Watchdog

AI Agentはデモで数分動けばよいものと、24時間常駐するものでは設計が変わります。

Hermes Agent Windowsでは、`scripts/windows/watchdog-go` に外部Go Watchdogを置いています。Desktop backend、local embedding service、関連runtimeなどの状態を監視します。

設計上重要なのは、**restart authorityを複数作らないこと**です。

Hermes本体、Electron、PowerShell、Goがそれぞれ勝手に再起動を始めると、障害時の挙動が予測不能になります。そのためforkでは、

> **外側のautomatic restart authorityはGo Watchdogだけ**

というルールを明示しています。

:::message
Go Watchdogの設計と接続安定化については「[zapabob × Hermes Agent：HermesDesktopwatchdogでWindows接続を安定させる](/articles/zapabob-hermesagent-watchdog-contributor)」も参照してください。
:::

---

## Upstreamをmergeするのではなくsemantic integrationする

Hermes Agent本体は非常に速い速度で開発されています。そのため単純に `git merge upstream/main` を繰り返す方式ではWindows固有機能が壊れやすくなります。

そこで現在はupstream SHAを固定し、upstream commitを以下に分類しています。

| 分類 | 意味 |
|------|------|
| **ADOPT** | そのまま採用 |
| **COMPOSE** | Windows downstreamのauthority modelと組み合わせて統合 |
| **DEFER** | 保留（プラットフォーム互換性の問題等） |
| **KEEP_DOWNSTREAM** | downstream固有実装を維持 |

最近のintegration campaignでは、

```text
Upstream delta commits : 1,049
ADOPT                  :   325
COMPOSE                :   723
DEFER_PLATFORM         :     1
Direct file intersection: 414
```

つまりupstreamをそのままコピーするのではなく、大半の変更をWindows downstreamのauthority modelと組み合わせています。

---

## Windows-firstだがWindows-onlyではない

repository名は `hermes-agent-windows` ですが、Hermes core自体はLinuxでも動きます。PythonのメインCIはUbuntu上で実行されています。

Windows固有なのは主に以下です。

- Go Watchdog / PowerShell lifecycle
- Windows packaging
- NTFS/process handling
- Windows-specific GPU/runtime qualification

実態としては、**Windows Native Tier-1 / Linux-compatible core** に近い構成です。

---

## GitHub Trafficを見たら予想外だった

最近GitHub Trafficを確認したところ、直近14日で次の数字が出ていました。

| 指標 | 数値 |
|------|------|
| Git clones | 9,808 |
| Unique cloners | 237 |
| Page views | 227 |
| Unique visitors | 80 |

Git cloneにはCIやbot、自動化も含まれる可能性があるため、237人が実際に利用したとは言えません。一方で、source-onlyでまだ正式Release assetも公開していない段階としては興味深い数字でした。

現在Starはまだ6です。実装規模に対してGitHub上で何を作っているrepositoryなのかが十分伝わっていなかった可能性もあり、今回この記事を書いてみることにしました。

---

## なぜ「Windows版Hermes」ではなく「AI Workstation」なのか

現在の構成をまとめるとこうなります。

```text
Hermes Agent Windows Workstation
│
├── Agent Runtime
│   ├── Sessions / Bots / MCP / Tools / Cron
│
├── Engineering
│   ├── Git Tree / Git CRUD / Diff / Review / Terminal
│
├── Browser
│   └── Integrated Web Workspace
│
├── Security
│   ├── ClamAV / YARA / Defender
│
├── Local AI
│   ├── llama.cpp / GGUF / Embeddings / Memory
│
├── Media
│   ├── Voice / TTS / VRChat / Unity
│
└── Operations
    ├── Go Watchdog / Recovery / Windows-native CI
```

なので最近は、

> Hermes AgentのWindows版を作っている

というより、

> **Hermes AgentをkernelとしてWindows AI Workstationを作っている**

と考えるようになりました。

---

## 今後

まだやりたいことはかなりあります。

- LSP diagnostics / symbol navigation
- test result visualization
- Local Model management UI
- Agent observability dashboard
- Security Center強化
- Linux compatibility documentation
- stable Windows installer / portable release

Hermes Agent upstreamの進化も非常に速いため、最新機能についてもmoving `main`を直接追うのではなく、immutable snapshotを使って継続的に統合していきます。

---

## Repository

**Hermes Agent Windows Workstation Edition**

https://github.com/zapabob/hermes-agent-windows

MIT Licenseです。

WindowsでLocal AI、Agent、Memory、Voice、VR、開発環境を一つにまとめたい人には、かなり面白い実験環境になってきたと思います。

StarやIssue、PRも歓迎です。

そして何より、

**「OS上でAIを使う」のではなく、「AIを中心にOS上の作業環境を組み直す」と何が起きるか。**

もう少しこの方向を掘ってみます。
