---
title: "OpenClawをRust+WASMで書き直し、RunPodのNemotron/Qwen3と繋いだら「完全自律AIエージェント」ができた"
emoji: "🦞"
type: "tech"
topics: ["rust", "wasm", "llm", "runpod", "openai"]
published: true
---

**TL;DR**: 247Kスターのオープンソースエージェント [OpenClaw](https://github.com/openclaw/openclaw) のgatewayをRust+WASMで書き直し、外部LLM APIに依存しないセルフホスト型のNVIDIA Nemotron-9B-v2とQwen3-32BをRunPodで動かして繋いだ話。OpenAIもAnthropicも使わない、完全自律AIの構築記録。

---

## なぜ「完全自律」なのか

OpenClawは「自分のデバイスで動かすパーソナルAIアシスタント」だ。WhatsApp・Telegram・Discord・Slack・iMessageなど既存のメッセージアプリを入力インターフェイスとして使い、ローカルに立てたgatewayがAIエージェントとして動く。

ここまでは多くのセルフホストAIと変わらない。違いは**どこで推論するか**だ。

```
通常のセルフホストエージェント:
[自分のサーバー/Mac] → [OpenAI API / Anthropic API]
                         ↑ここだけクラウド依存

完全自律スタック:
[自分のサーバー/Mac] → [RunPod vLLM: Nemotron/Qwen3]
  OpenClaw gateway       ↑オープンウェイト、自分のGPU
```

LLM推論もセルフホストにすると、外部APIへのトークン送信がゼロになる。プライバシー・コスト・レイテンシ、すべてが自分の管理下に入る。

---

## OpenClaw Rust化：なぜNode.jsをやめるのか

OpenClawは現在TypeScript/Node.jsで書かれている。動作するが、Node.jsランタイムへの依存がある。

```bash
# 現在の起動フロー
npm install -g openclaw@latest   # Node.js 22+ 必須
openclaw onboard --install-daemon
openclaw gateway --port 18789
```

これをRust+WASMに書き直すと何が変わるか：

| 項目 | TypeScript/Node.js | Rust binary | Rust → WASM |
|------|-------------------|-------------|-------------|
| 実行環境 | Node.js 22+ 必須 | OS直接実行 | ブラウザ / CF Workers / Wasmtime |
| バイナリサイズ | node_modules含め数百MB | ~5MB | ~3MB |
| 起動時間 | ~800ms | ~20ms | ~5ms |
| メモリ | ~80MB | ~8MB | ~4MB |
| クロスコンパイル | 困難 | 容易 | ターゲット追加のみ |

「どこでも動く」はRustのコンパイルターゲット多様性から来る：

```bash
cargo build --target aarch64-unknown-linux-musl   # Lambda / VPS
cargo build --target x86_64-unknown-linux-musl    # x86 Linux
cargo build --target wasm32-wasi                  # CF Workers / Wasmtime
cargo build --target wasm32-unknown-unknown       # ブラウザ
```

### Rust gatewayの構造

書き直したgatewayは4クレートで構成される：

```
openclaw/crates/
├── protocol/    # WebSocketフレーム型定義（JSON RPC）
├── config/      # JSON5設定ファイルパース
├── sessions/    # セッションストア（接続状態管理）
└── gateway/     # axum 0.8 + WebSocket プロキシサーバー
    └── src/
        ├── ws.rs      # WS handshake + bidirectional proxy (591行)
        ├── bridge.rs  # Node sidecar への中継
        ├── auth.rs    # 接続認証
        ├── sidecar.rs # プロセス管理
        └── server.rs  # axum ルーティング
```

WebSocket handshakeの実装（`ws.rs`）：

```rust
pub async fn handle_ws(mut socket: WebSocket, state: Arc<AppState>, req_ctx: RequestContext) {
    let conn_id = state.next_conn_id();
    match handshake(&mut socket, &state, &conn_tag, &req_ctx).await {
        Ok(()) => proxy_to_sidecar(&mut socket, &state, &conn_tag).await,
        Err(e) => warn!(conn = %conn_tag, ?e, "handshake failed"),
    }
}

async fn handshake(socket: &mut WebSocket, ...) -> Result<(), HandshakeError> {
    // Step 1: connect.challenge を送信
    let challenge = GatewayFrame::Event(EventFrame::new(
        "connect.challenge",
        Some(json!({ "nonce": nonce, "ts": now_ms })),
    ));
    send_frame(socket, &challenge).await?;

    // Step 2: connect リクエスト受信（タイムアウト付き）
    let msg = tokio::time::timeout(
        Duration::from_millis(HANDSHAKE_TIMEOUT_MS),
        receive_frame(socket)
    ).await??;

    // Step 3〜5: 認証 → hello-ok → Node sidecarへブリッジ
}
```

---

## RunPodでNemotron-9B-v2を動かす：エージェントとして使う現実

### Pod構成

```
GPU  : RTX A5000 (24GB VRAM)  $0.32/hr
Image: vllm/vllm-openai:latest-x86_64
Model: nvidia/NVIDIA-Nemotron-Nano-9B-v2-Japanese
Args : --trust-remote-code
       --max-num-seqs 32
       --max-model-len 8192
       --gpu-memory-utilization 0.90
       --dtype bfloat16
```

max-num-seqs=32にしているのは理由がある。最初は`max_model_len=131072`（128K）で試したが、KVキャッシュが巨大になりmax-num-seqs=4しか捌けず、複数ユーザーが使うと即座にキューが詰まった。

```
# KVキャッシュのトレードオフ
max_model_len=131072, max_num_seqs=4  → 長いが遅い（4同時）
max_model_len=8192,   max_num_seqs=32 → 短いが速い（32同時）
```

日常的なエージェント用途で8Kを超えるコンテキストは稀なので、スループット優先の8K設定が正解だった。

### Nemotronの推論フォーマット

Nemotronはreasoning modelなので、回答前に`<think>`タグ内で推論を展開する：

```
<think>
ユーザーは天気を聞いている。weather_fetchツールを呼ぶべき。
東京の座標は35.6762° N, 139.6503° E。
</think>

{weather_fetchの結果に基づいた回答}
```

ストリーミング実装での注意点：`enable_thinking: false`を送ると`</think>`タグが出てこなくなる。これに気づかず、`</think>`を待ってからコンテンツを流す実装にしていたら、全レスポンスが捨てられるバグが発生した。

```rust
// バグのあった実装
think_done = !is_runpod;  // RunPodでは think_done=false → 永久にバッファ

// 正しい実装
think_done = true;  // enable_thinking:false の場合、コンテンツを直接流す
```

### 実際のエージェント動作ログ

**天気確認タスク：**
```
User: 東京の今日の天気は？

[Tool Call] weather_fetch(location="東京")
[Tool Result] 晴れ、気温18℃、湿度55%

東京は現在晴れ、気温18℃です。週末にかけても晴天が続く見込みです。
```

**コード生成タスク：**
```
User: Rustでフィボナッチ数列を書いて

fn fibonacci(n: u32) -> u32 {
    match n {
        0 => 0,
        1 => 1,
        _ => fibonacci(n - 1) + fibonacci(n - 2),
    }
}
```

**ツール呼び出しの罠 — ツール名に敏感：**

Nemotronは学習データに少ないツール名を「利用できません」と返すことがある。

```
# これは呼ばなかった
web_fetch, qr_code  ← 学習データに少ない

# リネームしたら動いた
read_webpage, create_qr
```

8ファイル以上に影響する変更だったが、ツール名を揃えることで解決した。

---

## Qwen3-32B on RTX A6000：なぜA5000では3回失敗したか

### VRAMの壁

Qwen3-32B-AWQをA5000で動かそうとして3回OOMクラッシュした：

```
試み1: A5000, max_model_len=16384, max_num_seqs=4  → OOM crash loop
試み2: A5000, max_model_len=16384, max_num_seqs=1  → OOM crash loop
試み3: A5000, max_model_len=8192,  max_num_seqs=2  → OOM crash loop
```

クラッシュの症状：Podのアップタイムが-10秒〜+17秒でサイクル、CPU=0〜14%、HTTP 502。OOMによる再起動ループだ。

VRAMの計算：

```
Qwen3-32B-AWQ モデル重み  ≈ 16 GB
RTX A5000 VRAM            = 24 GB
KVキャッシュに使えるVRAM  ≈ 5.6 GB

KVキャッシュ per token = 2 × 64 layers × 8 KV-heads × 128 head-dim × 2 bytes
                       ≈ 0.25 MB/token

5,600 MB ÷ 0.25 MB/token ≈ 22,400 tokens（理論値）
実際は gpu_memory_utilization のマージンで ~4096 tokens が限界
```

A5000では4096トークンが限界。しかしシステムプロンプト（AGENT_COMMON + ツール説明 + メモリ）だけで**4789トークン**ある。**システムプロンプトだけでコンテキスト上限を超える**。

### RTX A6000 (48GB) で解決

```
GPU     : RTX A6000 (48 GB VRAM)  $0.49/hr
Model   : Qwen/Qwen3-32B-AWQ
Args    : --max-model-len 16384
          --max-num-seqs 4
          --gpu-memory-utilization 0.90
          --dtype bfloat16

VRAM内訳:
  モデル重み    ≈ 16 GB
  KVキャッシュ  ≈ 4 GB  (16K × 0.25MB/token)
  余剰          ≈ 23 GB  ← 余裕
```

### Qwen3の実際のレスポンス（Lambdaログより）

```
model_used    : "qwen3-32b"
input_tokens  : 4334
output_tokens : 159
credits_used  : 6

fn fibonacci(n: u32) -> u32 {
    match n {
        0 => 0, 1 => 1,
        _ => fibonacci(n-1) + fibonacci(n-2)
    }
}
```

Nemotronと比べて：
- **コンテキスト**: 8K → 16K（長い会話・ドキュメント処理に対応）
- **多言語**: 日本語・英語・中国語の混在に強い
- **推論品質**: 32Bパラメータの恩恵で複雑なロジックも正確

---

## 完全自律スタックの全体像

```
ユーザー（WhatsApp / Telegram / LINE / Discord）
  │
  ▼
OpenClaw Gateway（Rust binary / WASM）
  │  WebSocket + JSON RPC
  ▼
OpenClaw Agent（TypeScript → 順次Rust化）
  │
  ├── [web_search]   ──────────────→ Jina Reader
  ├── [read_webpage] ──────────────→ Jina Reader
  ├── [weather_fetch]──────────────→ Open-Meteo（無料）
  ├── [code_execute] ──────────────→ サンドボックス /tmp
  └── [LLM推論]      ──────────────→ RunPod vLLM
                                      ├── Nemotron-9B-v2 ($0.32/hr)
                                      └── Qwen3-32B-AWQ  ($0.49/hr)
```

**外部クラウドAPI依存ゼロ**の部分：
- OpenAI API → 使わない
- Anthropic API → 使わない
- LLM → RunPod上の自分のGPU（オープンウェイト）

RunPodのコスト試算：
- Nemotron-9B (A5000): ~$0.32/hr → 月$230（常時稼働）
- Qwen3-32B (A6000): $0.49/hr → 月$353（常時稼働）

同量をOpenAI GPT-4o APIで使うより安く、プライベートで、オフラインでも動く。

---

## WASMターゲットの意味：「どこでも動く」

Rust gatewayを`wasm32-wasi`にコンパイルすると：

```bash
# Cloudflare Workersで動く（エッジ、グローバル）
wrangler deploy --entry=gateway.wasm

# WasmEdgeで動く（Dockerすら不要）
wasmtime gateway.wasm --port 18789

# ブラウザで動く
# → Webワーカーとして常駐、サーバーレスAIエージェント
```

Node.js不要、Docker不要、OS依存なし。Raspberry Piでも、ルーターのWASMランタイム上でも動く。

これがユーザー獲得のストーリーになる：

| ユーザー | デプロイ方法 |
|---------|------------|
| 個人開発者 | `docker compose up` または `wasmtime gateway.wasm` |
| 企業 | CF Workersでエッジデプロイ、データが外に出ない |
| IoT/組み込み | ARMバイナリで低消費電力デバイスに常駐 |

---

## まとめ：自律AIの条件が揃ってきた

| レイヤー | 従来 | 今 |
|---------|------|-----|
| エージェント実行 | クラウドLLM API | セルフホスト（OpenClaw） |
| LLM推論 | OpenAI/Anthropic | Nemotron/Qwen3 on vLLM |
| バイナリポータビリティ | Node.js依存 | Rust binary / WASM |
| データ保存 | クラウドDB | ローカルSQLite / libSQL |

残る課題は**モデル品質**だ。9B・32Bでも多くのタスクを処理できるが、Claude OpusやGPT-4oクラスの推論能力はまだない。しかしNemotron・Qwen3の進化速度を見ると、差は急速に縮まっている。

オープンウェイトモデルとセルフホストインフラの組み合わせが成熟した今、「外部APIに頼らない自律AIエージェント」は夢ではなく、動くコードになりつつある。

---

**関連リンク**
- [OpenClaw GitHub](https://github.com/openclaw/openclaw) — 247K ⭐
- [chatweb.ai](https://chatweb.ai) — マルチモデルAIサービス（OpenClaw互換API）
- [RunPod](https://runpod.io) — GPU Pod ($0.32〜/hr)
- [vLLM](https://github.com/vllm-project/vllm) — OpenAI互換推論サーバー
- [Nemotron-9B-v2-Japanese](https://huggingface.co/nvidia/NVIDIA-Nemotron-Nano-9B-v2-Japanese)
- [Qwen3-32B-AWQ](https://huggingface.co/Qwen/Qwen3-32B-AWQ)
