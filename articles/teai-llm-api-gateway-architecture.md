---
title: "日本発のLLM API Gateway「teai.io」のアーキテクチャを全公開"
emoji: "🏗"
type: "tech"
topics: ["rust", "aws", "llm", "api", "lambda"]
published: true
---

## はじめに

[teai.io](https://teai.io) は、日本のAI開発者向けに設計されたLLM API Gatewayだ。45以上のモデルをOpenAI互換のAPIで提供し、東京リージョンで低レイテンシを実現している。

この記事では、teai.ioの技術的アーキテクチャを公開する。LLM Gatewayを自分で構築したい方や、似たようなサービスのアーキテクチャに興味がある方の参考になれば嬉しい。

## なぜ作ったか

LLMアプリ開発者として、こんな課題を感じていた：

- **OpenRouter** は素晴らしいが、サーバーが米国。日本から使うと200-400msの余計なレイテンシ
- ドキュメントが英語のみ
- 円建て請求・インボイス制度に非対応
- 日本のAI市場は$15.6B（2025年）で急成長中なのに、**日本向けのGatewayが存在しない**

「ないなら作ろう」で始めた。

## 全体アーキテクチャ

```
                          ┌─────────────────────┐
                          │   Cloudflare DNS     │
                          │   + CF Workers       │
                          │   (teai-edge)        │
                          └──────────┬───────────┘
                                     │
                          ┌──────────▼───────────┐
                   ┌──────│   AWS API Gateway     │
                   │      │   (ap-northeast-1)    │
                   │      └──────────┬───────────┘
                   │                 │
          fallback │      ┌──────────▼───────────┐
                   │      │   AWS Lambda (ARM64)  │
                   │      │   Rust (axum)         │
                   │      │   "nanobot-prod"      │
                   │      └──────────┬───────────┘
                   │                 │
          ┌────────▼──┐    ┌────────▼────────┐
          │  Fly.io   │    │    DynamoDB     │
          │  (nrt)    │    │  (sessions,    │
          │  fallback │    │   users, etc.) │
          └───────────┘    └────────┬────────┘
                                    │
                          ┌─────────▼─────────┐
                          │  LLM Providers     │
                          │  OpenAI / Anthropic│
                          │  Google / Groq     │
                          │  RunPod / DeepSeek │
                          └───────────────────┘
```

3層構造：
1. **エッジ (CF Workers)** — DNS解決、TLS終端、フォールバック制御
2. **コンピュート (Lambda)** — ルーティング、認証、クレジット計算、SSE中継
3. **ストレージ (DynamoDB)** — ユーザー、セッション、使用量

## なぜRust + Lambda？

### コールドスタートが速い

Rustバイナリは~5MBに収まり、Lambda上のコールドスタートが**50-100ms**。Node.js/Pythonの300-1000msと比べて圧倒的に速い。

API Gatewayは「常時起動するサーバー」ではなく「リクエストごとに計算して返す」パターンなので、Lambdaの従量課金モデルと相性が良い。

### メモリ効率

128MBのLambdaでも動作する（実際は256MB設定）。Rustのゼロコスト抽象化のおかげで、ランタイムコストが安い。

### 型安全性

45+モデルのルーティング、クレジット計算、認証をコンパイル時に型で保証できる。「GPT-4oのリクエストをAnthropicに投げてしまう」みたいなバグがコンパイル時に防げる。

### ビルドの罠：musl必須

Lambda (AL2023) で動かすには **musl ターゲットが必須**。gnuだとglibc互換性で `Runtime.ExitError` になる。

```bash
# 正しい
cargo zigbuild --target aarch64-unknown-linux-musl --release

# NG: 絶対にやらない
# cargo build --target aarch64-unknown-linux-gnu
```

これは実際に踏んだ地雷で、デバッグに半日かかった。Lambda + Rustの組み合わせでは必ずmuslを使おう。

## エッジ層: Cloudflare Workers

```javascript
// teai-edge Worker（簡略版）
export default {
  async fetch(request, env) {
    const url = new URL(request.url);

    // プライマリ: AWS Lambda
    try {
      const response = await fetch(env.PRIMARY_BACKEND + url.pathname, {
        method: request.method,
        headers: {
          ...Object.fromEntries(request.headers),
          "X-Forwarded-Host": url.hostname,  // ブランド判定用
        },
        body: request.body,
      });
      if (response.ok) return response;
    } catch (e) {
      // フォールバックへ
    }

    // フォールバック: Fly.io
    return fetch(env.FALLBACK_BACKEND + url.pathname, {
      method: request.method,
      headers: request.headers,
      body: request.body,
    });
  }
};
```

### なぜCF Workers？

- **レイテンシ**: CDNエッジでDNS解決〜TLS終端が完了。東京PoPから直接Lambdaへ
- **フォールバック**: Lambdaが落ちてもFly.ioに自動切替
- **コスト**: 無料プランで10万リクエスト/日。現状のトラフィックでは$0

## LLMプロバイダルーティング

teai.ioの核心は**プロバイダルーティング**。45+モデルを、それぞれ最適なプロバイダに振り分ける。

```rust
fn route_model(model: &str) -> Provider {
    match model {
        m if m.starts_with("gpt-") => Provider::OpenAI,
        m if m.starts_with("claude-") => Provider::Anthropic,
        m if m.starts_with("gemini-") => Provider::Google,
        m if m.contains("nemotron") => Provider::RunPod,
        m if m.starts_with("deepseek") => Provider::DeepSeek,
        m if m.contains("qwen") => Provider::Groq,
        _ => Provider::OpenAI,
    }
}
```

### フォールバックチェーン

各モデルには代替プロバイダが設定されている：

```rust
fn fallback_chain(model: &str) -> Vec<Provider> {
    match model {
        "gpt-4o" => vec![Provider::OpenAI, Provider::OpenRouter],
        "claude-sonnet-4-6" => vec![Provider::Anthropic, Provider::OpenRouter],
        "nemotron-9b" => vec![Provider::RunPod, Provider::Groq],
        _ => vec![Provider::OpenAI],
    }
}
```

プライマリが失敗したら、自動で次のプロバイダに切り替わる。ユーザー側のコード変更は不要。

## SSEストリーミング

LLM APIの多くはServer-Sent Events (SSE) でストリーミングレスポンスを返す。teai.ioはこれを**透過的に中継**する。

特にAnthropicは独自のSSEフォーマットを使うため、リアルタイムで変換が必要：

```
// Anthropic形式（受信）
event: content_block_delta
data: {"type":"content_block_delta","delta":{"type":"text_delta","text":"Hello"}}

// → OpenAI互換形式（送信）
data: {"choices":[{"delta":{"content":"Hello"},"index":0}]}
```

axumの `Sse` + tokioの非同期ストリームで、バッファリングなしの低レイテンシ中継を実現。

```rust
async fn handle_streaming(
    provider_response: Response<Body>,
) -> impl IntoResponse {
    let stream = provider_response
        .into_body()
        .into_data_stream()
        .map(|chunk| {
            let data = transform_to_openai_format(chunk?);
            Ok::<_, Error>(Event::default().data(data))
        });

    Sse::new(stream)
        .keep_alive(KeepAlive::default().interval(Duration::from_secs(15)))
}
```

## クレジットシステム

### 設計思想

- **透過的**: プロバイダの原価がそのまま見える
- **シンプル**: 1クレジット ≈ 特定のトークン数（モデルにより異なる）
- **リアルタイム**: レスポンスヘッダ `X-Credits-Remaining` で残高確認

```rust
struct ModelPricing {
    credits_per_1k_input: f64,
    credits_per_1k_output: f64,
}

fn calculate_cost(model: &str, input_tokens: u64, output_tokens: u64) -> f64 {
    let pricing = get_pricing(model);
    let input_cost = (input_tokens as f64 / 1000.0) * pricing.credits_per_1k_input;
    let output_cost = (output_tokens as f64 / 1000.0) * pricing.credits_per_1k_output;
    input_cost + output_cost
}
```

Nemotron 9B は `credits_per_1k_input: 0.0` で完全無料。開発者がリスクゼロで試せるようにしている。

## パフォーマンス

実測値（東京→東京、2026年3月時点）：

| メトリクス | 値 |
|-----------|-----|
| DNS解決 | ~5ms (CF) |
| TLS接続 | ~10ms (CF edge) |
| Edge → Lambda | ~15ms |
| Lambda コールドスタート | ~80ms |
| Lambda ウォームスタート | ~5ms |
| **合計オーバーヘッド** | **~35-110ms** |

LLMの推論自体が500ms-5sかかるため、35-110msのオーバーヘッドはほぼ無視できる。

OpenRouterは米国サーバー経由で200-400msのオーバーヘッドがあるので、**日本からのアクセスではteai.ioの方が確実に速い**。

## インフラコスト

月間1万リクエスト想定：

| コンポーネント | 月額 |
|---------------|------|
| CF Workers | $0（無料枠内） |
| AWS Lambda (256MB) | ~$0.50 |
| API Gateway | ~$3.50 |
| DynamoDB | ~$1.00 |
| **合計** | **~$5.00** |

トラフィックが増えても、Lambdaの従量課金のおかげでコストはリニアにスケール。固定費がほぼゼロ。

## 学んだ教訓

### 1. musl vs gnu — Lambdaでは必ずmusl
gnuビルドはLambdaで `Runtime.ExitError` になる。半日溶かした。

### 2. include_str!() のキャッシュ問題
HTMLをバイナリに埋め込む `include_str!()` は、ソース変更だけでは再ビルドされないことがある。`cargo clean` が必要な場合がある。

### 3. API Gatewayの$LATEST直接呼び出し
API Gatewayは `$LATEST` を直接呼ぶ設定にしているため、`update-function-code` した瞬間にプロダクション影響。カナリアデプロイが事実上できない。

### 4. SSEのアイドルタイムアウト
ストリーミング中のアイドルタイムアウトは30秒。O3のような長い推論ではkeep-aliveが必須。

## OpenAI互換APIの実装

開発者にとって最も重要なのは「既存コードが動くか」。teai.ioはOpenAI APIと完全互換のエンドポイントを提供：

```python
from openai import OpenAI

client = OpenAI(
    base_url="https://api.teai.io/v1",  # この1行だけ変更
    api_key="te_your_api_key"
)

# GPT-4o, Claude, Gemini... 何でも同じインターフェースで
response = client.chat.completions.create(
    model="claude-sonnet-4-6",
    messages=[{"role": "user", "content": "Hello!"}],
    stream=True
)
```

Tool Calling、ストリーミング、システムメッセージ — すべてOpenAI SDKの仕様に準拠。

## おわりに

teai.ioのアーキテクチャは、**Rust + Lambda + CF Workers** というシンプルな構成で、低コスト・低レイテンシ・高可用性を実現している。

日本のAI開発者コミュニティにとって、為替リスクなし・インボイス対応・日本語サポートの選択肢があることは価値があると信じている。

**無料で1,000クレジット + Nemotron 9B無制限**なので、興味があればぜひ → [teai.io](https://teai.io)

フィードバックや質問はコメントでどうぞ。
