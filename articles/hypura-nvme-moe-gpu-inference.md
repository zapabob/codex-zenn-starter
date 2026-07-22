---
title: "VRAMに乗らないMoEをNVMe＋GPU推論で動かす：Hypura/llama.cpp/TurboQuant解説"
emoji: "⚡"
type: "tech"
topics: ["llm", "cuda", "gguf", "moe", "llmops"]
published: true
---

> **一次資料**
> - [zapabob/hypura](https://github.com/zapabob/hypura) — ストレージ階層対応GGUFランタイム
> - [zapabob/llama.cpp](https://github.com/zapabob/llama.cpp) — SO(8) Triality / TurboQuant KVキャッシュ研究fork
> - [zapabob/Turboquant-CUDA](https://github.com/zapabob/Turboquant-CUDA) — KVキャッシュ量子化研究・GGUF生成ツールチェーン

---

## なぜこの問題が起きるのか

Qwen3-235B や DeepSeek V3 などのSparse MoEモデルは、全パラメータのうち実際に推論で使うのは**トークンあたり数エキスパートだけ**です。しかし、モデル全体のGGUFファイルは20〜200GB以上になります。

コンシューマGPU（RTX 5060 Ti / VRAM 8〜16GB）にはとても乗りません。

従来の対処法と問題点：

| 対処法 | 問題 |
|--------|------|
| CPU オフロード（`-ngl 0`） | GPU CUDA コアが遊ぶ。1〜2 tok/s が限界 |
| OS スワップ（NVMe 書き込み） | ランダム書き込みでNVMe寿命を消耗。10〜30倍遅くなる |
| 小さいモデルに変える | 品質が落ちる |

zapabob の3リポジトリは、この問題を**「GGUFをNVMeに読み取り専用で置き、活性化されたエキスパートだけGPUで処理する」**という設計で解決します。

---

## 3リポジトリの役割と関係

```
zapabob/Turboquant-CUDA
  ↓ TQ4_1S GGUF / Triality schema v2 を生成・検証
zapabob/llama.cpp
  ↓ 低レベルGGUF解析・SO(8)検証・CUDAカーネル・C ABI
zapabob/hypura
  ↓ ストレージ階層スケジューラ（GPU / ピン留めRAM / ページRAM / NVMe）
     → HTTP API / KoboldCpp互換サーバ
```

### zapabob/llama.cpp：低レベル基盤

本家 `ggml-org/llama.cpp` のresearchfork。追加された主な機能：

- **TurboQuant KVキャッシュ圧縮**（`TQ4_1S` / `TQ3_1S`）
- **SO(8) Triality K側保護**（`key_only_block_so8_triality_vector`）
- **KV精度評価ループ**（単なるMSE再構成ではなく、隠れ状態コサイン類似度・アテンション相対誤差・記憶比率で選定）

ablation結果（RTX 3060 12GB、Qwen3.5-9B）：

| KVモード | 隠れ状態コサイン | アテンション相対誤差 | 記憶比率 |
|----------|-----------------|---------------------|---------|
| `exact` | 1.000000 | — | 1.000 |
| `key_only_block_so8_triality_vector` | **0.999349** | — | **0.629** |
| `asym_q8_turbo4` | 0.994141 | 0.149698 | 0.379 |
| `full_kv` | 0.994792 | — | 0.256 |

`key_only_block_so8_triality_vector` が**品質を保ちながらKVキャッシュを37%削減**するプロダクション推奨ラインです。

### zapabob/Turboquant-CUDA：GGUFスキーマ生成・検証

TurboQuant研究の検証ツールチェーン。

- `TQ4_1S` → GGUF変換（バイト完全一致）
- Triality schema v2 の束バンドルメタデータ・マニフェストハッシュ生成
- RTX 3060 12GBでのリアルハードウェア検証（Qwen3.5-9B / Gemma 4）
- NC-KA（数値ランク・条件付け）・URT（再現可能変換記録）ディスクリプタ

```bash
# 検証コマンド（uv使用）
uv sync --extra cu128 --extra dev --extra hf_qwen --extra eval
uv run python scripts\env_check.py
uv run python scripts\validate_repo_contract.py
```

### zapabob/hypura：4階層ストレージスケジューラ

Hypura v1.0.0 の核心は**Triality Council実行パス**と**4階層テンソル配置**です。

```
┌─────────────────────────────────────────────┐
│  VRAM (GPU)                                  │
│  アテンション層・ノルム・埋め込み（常時使用）      │
├─────────────────────────────────────────────┤
│  ピン留めホストメモリ（CPU pinned）             │
│  頻繁に使うエキスパート（PCIe転送高速化）         │
├─────────────────────────────────────────────┤
│  ページRAM                                   │
│  使用頻度の中程度のエキスパート                  │
├─────────────────────────────────────────────┤
│  NVMe（読み取り専用mmap）                      │
│  GGUFファイル全体 / 低頻度エキスパート           │
└─────────────────────────────────────────────┘
```

**重要：NVMeへの書き込みは発生しません。** `mmap`は読み取り専用操作です。OSのページキャッシュがヒットすればNVMeアクセス自体も起きません。

---

## 実測ベンチマーク（Qwen3.6-35B-A3B、RTX 5060 Ti、2026-07-16）

Hypura READMEから直接引用：

> モデルサイズ 19.7GB が RTX 5060 Ti の VRAM を超えるため、ランタイムはモデルウェイトをCPUに保持（`n_gpu_layers=0`、`CPU_Mapped` 約 20,175.71 MiB）し、CUDAはコンピュートバッファのみに使用。

| 実行モード | プロンプト評価 | デコード | 壁時計時間 | 配置 |
|-----------|--------------|---------|------------|------|
| llama.cpp ベースライン | 1.8 tok/s | 1.4 tok/s | 23.4秒 | GPU 0層（ランタイム拒否後） |
| Hypura `legacy-3tier` + pinned off | 4.6 tok/s | **5.3 tok/s** | 10.0秒 | Sparse-MoE mmap、CPUウェイト |
| Hypura `four-tier` + pinned off | 0.7 tok/s | 2.8 tok/s | 39.7秒 | Sparse-MoE mmap、CPUウェイト |
| Hypura `four-tier` + pinned auto | 2.2 tok/s | **4.2 tok/s** | 12.0秒 | Sparse-MoE mmap、CPUウェイト |

**`four-tier` + `auto` はベースラインの3.10倍のデコード速度**（8トークンスモーク測定）。

`legacy-3tier`（5.3 tok/s）は短いコンテキストの1セッションなら**実用的な速度**と評価されています。`four-tier` + `auto`（4.2 tok/s）はより保守的なメモリ配置で安定性重視のモードです。

---

## なぜNVMeへの書き込みが不要なのか

この点は誤解が多いため整理します。

```
❌ 誤解：Hypuraはモデルをスワップ（NVMe書き込み）する
✅ 正解：GGUFをNVMe上でmmap（読み取り専用）し、必要なページだけOSがRAMに読み込む
```

`mmap` の動作：

1. GGUFファイルをOSの仮想アドレス空間にマッピング
2. アクセスされたページのみを物理RAM上に読み込む（ページフォルト）
3. RAMが足りなくなればOSが古いページを解放（書き込みは不要）
4. NVMeはあくまで**読み取りキャッシュのバックエンド**

Sparse MoEモデルでは、各トークンで活性化するエキスパートが全体の数%以下です。時間的局所性（同じエキスパートが連続して使われる傾向）により、**OSのページキャッシュヒット率が高く**、NVMeへのアクセス頻度は想定より低くなります。

---

## セットアップ概要

### 必要な環境

- Windows 10/11（Hypura v1.0.0はWindows binary対象）
- NVIDIA RTX 50シリーズ（CUDA 12.8 / compute `sm_120`）
- NVMe SSD（モデルGGUFを格納）
- 十分なRAM（最低モデルの50〜70%推奨）

### 手順

```powershell
# 1. GGUFをNVMeに配置
#    例: D:\models\Qwen3.6-35B-A3B-Uncensored-Q4_K_M.gguf

# 2. Hypura起動（four-tier + auto モード）
.\Hypura.exe serve `
  --model D:\models\Qwen3.6-35B-A3B-Uncensored-Q4_K_M.gguf `
  --storage-profile four-tier `
  --pinned auto `
  --turboquant-mode exact `
  --port 8080

# 3. OpenAI互換エンドポイントで確認
curl http://localhost:8080/v1/chat/completions `
  -H "Content-Type: application/json" `
  -d '{"model":"local","messages":[{"role":"user","content":"Hello"}]}'
```

---

## Triality Council：3候補から最良を選ぶ

Hypura v1.0.0 の新機能。同一プロンプトを3つのTriality視点で評価し、決定論的な3×3クロススコアリングで最良の応答を選びます。

```
Prompt
  ├── Triality View A → 候補A
  ├── Triality View B → 候補B
  └── Triality View C → 候補C
          ↓
    3×3 クロススコアリング（teacher-forced）
          ↓
    最良候補を選択（ログに選択理由・スコア行列を記録）
```

並列実行は、設定したVRAMヘッドルームを確保できる場合のみ `auto` で許可。メモリが足りなければフェイルクローズド（安全側で停止）します。

---

## TurboQuant：KVキャッシュを最大82%削減

KVキャッシュはLLM推論のVRAM消費の主要因の一つです。TurboQuant（Turboquant-CUDA研究で開発）はこれを圧縮します。

```
exact  KV（基準）      記憶比率 1.000  隠れ状態コサイン 1.000
 ↓ TurboQuant適用
full_kv（4bit相当）   記憶比率 0.256  隠れ状態コサイン 0.995
asym_q8_turbo4        記憶比率 0.379  隠れ状態コサイン 0.994  ← 積極的節約ライン
key_only_so8          記憶比率 0.629  隠れ状態コサイン 0.999  ← プロダクション推奨
```

K側はSO(8) Triality直交変換で数値安定性を保ち、V側は通過した品質閾値の中で最小ビット数（3bit）を選択します。

---

## まとめ：何が変わるか

| 状況 | 従来 | Hypura + TurboQuant |
|------|------|----------------------|
| 35B MoEモデル / 8GB VRAM | 動かない or 1.4 tok/s | **4〜5 tok/s**（3倍超） |
| NVMe書き込み寿命 | スワップで消耗 | **読み取りのみ（消耗なし）**|
| KVキャッシュメモリ | 100% | **25〜63%**（TurboQuant） |
| 推論品質（隠れ状態コサイン） | 1.000 | **0.999〜0.994**（劣化小）|

「VRAMに乗らないから諦める」ではなく、**ストレージ階層を推論ティアとして設計する**という考え方が、zapabob の3リポジトリが示すアプローチです。

---

## 参考リンク

- [zapabob/hypura](https://github.com/zapabob/hypura)
- [zapabob/llama.cpp](https://github.com/zapabob/llama.cpp)
- [zapabob/Turboquant-CUDA](https://github.com/zapabob/Turboquant-CUDA)
- [llama.cpp upstream（ggml-org）](https://github.com/ggml-org/llama.cpp)
