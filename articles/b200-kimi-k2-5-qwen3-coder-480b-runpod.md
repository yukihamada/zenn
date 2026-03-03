---
title: "RunPod 4xB200で1兆パラメータMoE「Kimi K2.5」と「Qwen3-Coder-480B」を同時に動かしてみた"
emoji: "🚀"
type: "tech"
topics: ["llm", "runpod", "vllm", "gpu", "moe"]
published: true
---

## はじめに

最近のオープンウェイトLLMの巨大化が止まらない。Kimi K2.5は1兆パラメータ、Qwen3-Coder-480Bは480Bパラメータ。どちらもMoEアーキテクチャで、推論時のアクティブパラメータは32B〜35B程度に抑えられているとはいえ、重みファイルだけでそれぞれ630GB、480GBある。

「これ、動かせるのか？」

最初は8xB200（VRAM 1,440GB）を狙っていたが、RunPodで在庫切れ。仕方なく4xB200（VRAM 720GB）で挑戦することにした。結果的に、1台ずつなら余裕で載ることがわかり、2つのポッドに分けて同時起動する構成に落ち着いた。

## なぜこの2モデルか

**Kimi K2.5** は、Moonshotが公開した現時点で最大クラスのオープンウェイトMoEモデルだ。総パラメータ1兆、エキスパート数384。INT4量子化で630GBに圧縮されているが、それでも常識的なGPU構成では動かせない規模感がある。マルチモーダル対応で、ツール呼び出しにも対応している。

**Qwen3-Coder-480B** は、Alibabaが公開したコーディング特化のMoEモデル。480Bパラメータ、160エキスパート、アクティブ35B。FP8で480GB。コード生成・リファクタリング・デバッグに振り切った設計だ。

両方ともMoEなので、パラメータ数の割に推論コストが抑えられる。「巨大だが実用的」という新しいカテゴリのモデルを実際に動かすのが今回の趣旨だ。

## VRAM計算

NVIDIA B200は1枚あたり180GB VRAM。4枚で720GB。

| モデル | 精度 | 重みサイズ | 4xB200(720GB)で動くか |
|--------|------|----------|---------------------|
| Kimi K2.5 | INT4（公式） | 630GB | ギリギリ動く（余裕90GB） |
| Qwen3-Coder-480B | FP8 | 480GB | 余裕あり（余裕240GB） |

Kimi K2.5はINT4で630GBなので、720GBのうち約90GBがKVキャッシュに使える。`max-model-len`を16384に絞り、`gpu-memory-utilization`を0.88にすることでOOMを回避できる見込みで設定した。

Qwen3-Coder-480BはFP8で480GB。約240GBの余裕があるので`max-model-len`を32768まで伸ばせた。

## RunPodでの起動手順

**Kimi K2.5（Pod ID: `6875dto8gg07u8`）:**

```bash
vllm serve moonshotai/Kimi-K2.5 \
  --tensor-parallel-size 4 \
  --trust-remote-code \
  --max-model-len 16384 \
  --gpu-memory-utilization 0.88 \
  --served-model-name kimi-k2.5 \
  --mm-encoder-tp-mode data \
  --tool-call-parser kimi_k2 \
  --enable-auto-tool-choice \
  --dtype auto
```

`--tensor-parallel-size 4`で4枚のB200にモデルを分割配置。`--trust-remote-code`はKimiのカスタムアーキテクチャに必要。`--tool-call-parser kimi_k2`でKimi K2系のツール呼び出しフォーマットに対応する。

**Qwen3-Coder-480B FP8（Pod ID: `q2jvcudcatcgiw`）:**

```bash
VLLM_USE_DEEP_GEMM=1 vllm serve Qwen/Qwen3-Coder-480B-A35B-Instruct-FP8 \
  --tensor-parallel-size 4 \
  --enable-expert-parallel \
  --max-model-len 32768 \
  --gpu-memory-utilization 0.88 \
  --served-model-name qwen3-coder-480b \
  --enable-auto-tool-choice \
  --chat-template-kwargs enable_thinking=false
```

`VLLM_USE_DEEP_GEMM=1`はFP8演算のカーネル最適化。`--enable-expert-parallel`でMoEのエキスパート並列とテンソル並列を組み合わせる。

最大の待ち時間はモデルDL。630GB（Kimi K2.5）は30〜60分かかる。この間も$19.96/hrが流れ続けるので、HuggingFaceの転送速度がコストに直結する。

## 起動確認方法

```bash
# ヘルスチェック
curl https://{pod_id}-8000.proxy.runpod.net/health

# OpenAI SDK経由
from openai import OpenAI
client = OpenAI(
    base_url="https://{pod_id}-8000.proxy.runpod.net/v1",
    api_key="dummy"
)
response = client.chat.completions.create(
    model="kimi-k2.5",
    messages=[{"role": "user", "content": "Rustでクイックソートを書いて"}]
)
```

vLLMログで確認すべきポイント：
- `GPU blocks: XXX` → 0だとOOM
- `Uvicorn running on http://0.0.0.0:8000` → 起動完了

## テスト結果

:::message
**2026年3月3日 現在両ポッドともモデルDL中のため随時追記**

**Kimi K2.5（4xB200, INT4）**
- 日本語チャット品質：追記予定
- マルチモーダル（画像入力）：追記予定
- ツール呼び出し安定性：追記予定
- TTFT / トークン毎秒：追記予定

**Qwen3-Coder-480B（4xB200, FP8）**
- コード生成品質（Kimi K2.5との比較）：追記予定
- 32Kコンテキスト活用例：追記予定
- TTFT / トークン毎秒：追記予定
:::

## コスト計算

| 期間 | コスト（2ポッド合計） |
|------|---------------------|
| 1時間 | $39.92（約6,000円） |
| 8時間作業 | $319（約48,000円） |
| 常時稼働（月） | $28,742（約430万円） |

常時稼働は非現実的。テスト・ベンチマーク目的で数時間動かして即停止が正解。RunPodはポッド停止でGPU課金が止まる（ストレージのみ継続）。

## まとめ

1兆パラメータのモデルがオープンウェイトで公開され、クラウドGPU4枚で動く時代になった。

MoEの恩恵で推論時のアクティブパラメータは32B〜35B。重みの格納にVRAMが必要なだけで、実際の計算量は30B級モデルと大差ない。「巨大だが実用的」がMoEの本質だ。

8xB200が在庫切れで4枚構成になったのは誤算だったが、計算したら4枚で十分だと判明。制約が「本当に必要な最小構成」を考えさせてくれた。テスト結果は随時追記する。

---

**関連リンク**
- [moonshotai/Kimi-K2.5](https://huggingface.co/moonshotai/Kimi-K2.5)
- [Qwen/Qwen3-Coder-480B-A35B-Instruct-FP8](https://huggingface.co/Qwen/Qwen3-Coder-480B-A35B-Instruct-FP8)
- [RunPod B200](https://www.runpod.io/gpu-models/b200) — $4.99/hr/GPU
- [vLLM Kimi K2.5 Recipe](https://docs.vllm.ai/projects/recipes/en/latest/moonshotai/Kimi-K2.5.html)
- [前シリーズ：OpenClawをRust+WASMで書き直した話](https://zenn.dev/yukihamada/articles/openclaw-rust-wasm-runpod-nemotron-qwen3)
