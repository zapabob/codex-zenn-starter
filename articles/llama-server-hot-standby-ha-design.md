---
title: "llama-serverのホットスタンバイは存在するか：部品は揃っているのに名前がない"
emoji: "🔥"
type: "tech"
topics: ["llm", "llamacpp", "selfhosted", "highavailability", "aiagent"]
published: true
---

## はじめに：よくある混同

ローカルLLMを常用していると、「ホットスタンバイ」「モデル常駐」「KVキャッシュ保持」「スリープ復帰」「ホットスワップ」という言葉が混同されがちです。

結論から言うと、`llama-server` にホットスタンバイという名前の完成された単一機能はありません。しかし、**複数の機能を組み合わせるとホットスタンバイ構成を作れる**状態にはなっています。

この記事では、何がホットスタンバイで何がそうでないかを整理し、ローカルAIエージェント（Hermes Agent等）の可用性設計に応用する視点を示します。

---

## llama-server が持っている部品

現在の upstream `llama-server` には、以下の機能が揃っています。

- **ルーターモード**：モデルを指定せず起動し、要求のモデル名に従って動的にロード・アンロード
- **`load-on-startup`**：プリセットでモデルを起動時からロード済みにする
- **モデル状態管理**：`/models` で `loaded` / `loading` / `sleeping` / `unloaded` を確認
- **ロード・アンロードAPI**：`/models/load` と `/models/unload`
- **ヘルスチェック**：`/health` はモデルロード中に503、利用可能時に200
- **スロット空き確認**：`/slots?fail_on_no_slot=1` で空きがなければ503

公式ドキュメントではこれらを**モデル管理機能**として記載しています。高可用性やactive–passive構成としてまとめた説明は、2026年8月時点でもありません。

> 参考：[llama.cpp server README](https://github.com/ggml-org/llama.cpp/blob/master/tools/server/README.md)

---

## 混同されやすい3つの構成

| 構成 | 実態 | 障害耐性 |
|------|------|---------|
| 単一サーバーでモデルをロード済みにする | 待機時間なしで推論開始 | プロセスやGPU障害には弱い |
| `-np` による複数スロット | 同じモデル内の並列コンテキスト | レプリカではない |
| **複数の独立サーバーをロード済みにする** | **本来のホットスタンバイ** | プロキシ側の切替で障害回避可能 |

`-np 4` で4スロットを確保しても、それは1つのプロセス内の並列処理です。プロセスが落ちれば全スロットが消えます。**スロット数を増やすことは冗長化ではありません。**

---

## `--sleep-idle-seconds` はホットスタンバイではない

ここが最も誤解されやすい部分です。

`--sleep-idle-seconds` は、アイドル状態が続いたとき、モデル本体とKVキャッシュを含む関連メモリをRAMからアンロードし、次の推論要求で再ロードする**省資源機能**です。

```text
                    ┌─────────────────────────┐
 --sleep-idle-seconds  │  loaded → sleeping → loaded  │
                    │  （復帰時にモデル再ロード発生）    │
                    └─────────────────────────┘
```

復帰にはモデルロード時間がかかるため、これは**ホットスタンバイとは反対側**の動作です。コールド復帰、あるいはウォーム寄りのスタンバイと呼ぶのが正確です。

一方で、`/health`・`/props`・`/models` はモデルがスリープ中でもモデルを起こさずに応答します。つまり**監視だけならスリープ中でも可能**です。

---

## 本物のホットスタンバイ構成

真のホットスタンバイは、同一モデルを**複数の独立プロセスでロード済みにしておく**ことで成立します。

```text
Inference Gateway（HAProxy / nginx / 独自supervisor）
    │
    ├── primary   llama-server :8080
    │              GPU0, model loaded, /health → 200
    │
    └── standby   llama-server :8081
                   GPU1, model loaded, /health → 200
```

ゲートウェイがprimaryのタイムアウト・接続拒否・異常終了・連続5xxを検出した時点で、standbyへ送信先を切り替えます。standby側はすでにモデルをRAM・VRAMへ展開済みなので、**モデルロード時間を待たずに**次の要求を処理できます。

コミュニティでも、複数の `llama-server` をロードバランサーの背後に置き、`/health` やスロット数を基に振り分ける運用は以前から行われています。

> 参考：[`server` production readiness · Discussion #6398](https://github.com/ggml-org/llama.cpp/discussions/6398)

ただし、同一モデルの複数インスタンスを一つのモデル名として扱い、優先順位付きで自動振り分けする機能は、2026年6月時点でも要望として議論段階でした。つまり、**完全なactive–passive HAが `llama-server` 本体で一つの完成機能になっているわけではありません。**

> 参考：[Shared Alias for multiple instances · Discussion #22823](https://github.com/ggml-org/llama.cpp/discussions/22823)

---

## フェイルオーバー時に失われるもの

`llama-server` を二重化しただけでは**会話状態は引き継がれません。**

KVキャッシュは各サーバー固有です。フェイルオーバー後は以下が必要になります。

1. **メッセージ履歴の再送**：Gateway側に保存した会話履歴をstandbyへ送り直す
2. **Prefillのやり直し**：standby側でKVキャッシュを再構築する
3. **進行中の生成の喪失**：ストリーミング途中の生成は基本的に失われる
4. **Gateway側の状態管理**：要求ID、再試行回数、重複生成防止、部分ストリーム破棄

つまり、推論サーバーの冗長化には**モデル以外の状態管理**がセットで必要です。

---

## AIエージェント構成での応用：Generation plane と Memory plane

Hermes Agentのような長期稼働エージェントでは、推論とメモリを分離した構成が自然に出てきます。

```text
Agent Gateway
├── Generation plane（生成系）
│   ├── CUDA primary     llama-server :8080
│   └── CUDA standby     llama-server :8081
│
└── Memory plane（記憶系）
    ├── embedding primary   llama-server :8090
    └── embedding standby   llama-server :8091
```

生成モデルは一時停止しても再試行できます。しかし、**埋め込みサーバーが停止すると、記憶の保存・検索・再ランキング・グラフ更新がまとめて止まります。**

したがって、埋め込み系のホットスタンバイは生成系より価値が高い場面があります。

### 埋め込みサーバーの同一性保証

ここに独自の難しさがあります。primaryとstandbyで以下を**完全に一致**させる必要があります。

```yaml
model_sha256:        identical  # GGUFファイルのハッシュ
gguf_quantization:   identical  # 量子化方式（Q4_K_M等）
embedding_dimension: 1024       # 次元数
pooling:             identical  # CLS / mean 等
normalization:       identical  # L2正規化の有無
distance_metric:     identical  # cosine / dot / L2
runtime_semantics:   compatible # llama.cpp バージョン
```

単に「同系列モデルを2つ置く」だけでは不十分です。モデル・量子化・pooling・正規化・ランタイム実装のどれかが異なると、切替前後で**ベクトル空間が変わり、既存インデックスに対する類似度の意味が静かに崩れます。**

---

## なぜ知られていないのか

| 理由 | 詳細 |
|------|------|
| 公式の説明が分散している | 推論API、スロット、ルーターモード、モデル管理に機能が散在 |
| 名前がない | 「ホットスタンバイ」として説明されていない |
| 単一GPU運用が主流 | 一般的なローカルLLM利用者はモデル二重化を想定しない |
| メモリの二重化が必要 | ホットスタンバイにはRAMまたはVRAMを倍量必要とする |
| `-np` との混同 | スロット数を冗長性と誤解しやすい |
| ベンチマークの文化 | 速度と量子化率が中心で、可用性設計はあまり扱われない |
| 完成にはプロキシ設計が必要 | 再試行、ストリーム途中切断、KVキャッシュの扱いまで設計しないと成立しない |

---

## まとめ

`llama-server` のホットスタンバイは、隠れた小技というよりも、**部品はすでに相当揃っているのに、HA設計として名前と標準構成が与えられていない機能群**です。

| 機能 | 状態 |
|------|------|
| ヘルスチェック（`/health`） | ✅ 利用可能 |
| モデル状態管理（loaded / sleeping） | ✅ 利用可能 |
| OpenAI互換API | ✅ 利用可能 |
| ルーターモード（動的ロード） | ✅ 利用可能 |
| 同一モデルの複数レプリカHA | ❌ 上位層（Gateway / LB）で構築 |
| KVキャッシュの引き継ぎ | ❌ フェイルオーバー後は再構築 |

ローカルAIエージェントを常用サービスとして運用する段階では、推論速度を少し高めるより、**この可用性設計のほうがシステム全体の信頼性に大きく関わります。**

---

## 参考リンク

- [llama.cpp server README](https://github.com/ggml-org/llama.cpp/blob/master/tools/server/README.md)
- [server production readiness · Discussion #6398](https://github.com/ggml-org/llama.cpp/discussions/6398)
- [Shared Alias for multiple instances · Discussion #22823](https://github.com/ggml-org/llama.cpp/discussions/22823)
