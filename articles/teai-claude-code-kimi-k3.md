---
title: "Claude CodeからKimi K3を直結で使う — teai.ioにAnthropic Messages API互換を自作した話"
emoji: "🔌"
type: "tech"
topics: ["claude", "llm", "api", "rust", "anthropic"]
published: true
---

## はじめに

[teai.io](https://teai.io) は日本発のLLM API Gatewayで、OpenAI互換のエンドポイントで85以上のモデルを提供している（アーキテクチャの詳細は[前回の記事](https://zenn.dev/yukihamada/articles/teai-llm-api-gateway-architecture)を参照）。

今回、[Claude Code](https://claude.com/claude-code)（Anthropic公式のコーディングエージェントCLI）から、teai.io経由でKimi K3をはじめとするカタログ上の任意モデルに、プロキシなしで直結できるようにした。

## 課題: Claude CodeはOpenAI互換を話せない

teai.ioはこれまでOpenAI Chat Completions形式（`/v1/chat/completions`）で他社互換を実現してきた。OpenAI SDKを使うアプリなら `base_url` を変えるだけで動く。

ところがClaude Codeは違う。内部的に `POST /v1/messages`（Anthropic Messages API）という別フォーマットしか話さない。`ANTHROPIC_BASE_URL` を任意のエンドポイントに向けられる設計にはなっているが、そのエンドポイントがMessages API互換でなければ動かない。これまではClaude CodeからOpenRouter経由のKimi K3を使うには、変換プロキシを別途自分で立てるしかなかった。

## 実装方針: 変換はHTTPハンドラ層で完結させる

既存の `LoadBalancedProvider`（OpenAI Chat Completions形式でプロバイダにリクエストを投げる内部コンポーネント）はそのまま再利用し、`POST /v1/messages` のハンドラ内でリクエスト/レスポンスをAnthropic形式⇔OpenAI形式に変換する構成にした。ルーティングやフォールバック、クレジット計算のロジックには手を入れていない。

### tool_use ⇔ tool_calls の相互変換

一番のポイントはここ。Anthropicの `tool_use` ブロックとOpenAIの `tool_calls` はデータの持ち方が違う。

- Anthropic `input_schema` ⇔ OpenAI `parameters`
- アシスタント側の `tool_use` ブロック ⇔ `tool_calls` 配列
- ユーザー側の `tool_result` ブロック ⇔ `role: "tool"` の独立メッセージ

これを双方向で実装し、`stop_reason` も `end_turn` / `tool_use` / `max_tokens` にマッピングした。マルチターンのtool往復（呼び出し→結果返却→次の応答）が壊れると、Claude Codeはコード編集や検索といった基本動作ができなくなるため、ここは特に慎重にテストした。`content` は単純な文字列とブロック配列（`text`/`tool_use`/`tool_result`）の両方をパースできるようにしている。

### SSEイベント順序の再現

ストリーミング時、Anthropic Messages APIは独自のイベント順序を持つ。

```
message_start
  → content_block_start
  → content_block_delta (text_delta | input_json_delta)
  → content_block_stop
  → message_delta
  → message_stop
```

OpenAI形式の単純なdelta羅列とは構造が異なるため、内部で受け取ったOpenAI形式のストリームを、このイベント順序に沿って再構築するレイヤーを書いた。ツール呼び出しの引数（JSON）が分割送信される場合は `input_json_delta` として逐次流している。

### 認証と課金

Claude Codeは `ANTHROPIC_AUTH_TOKEN` を `Authorization: Bearer` ではなく `x-api-key` ヘッダで送ってくるため、両方を受理するようにした（`anthropic-version` ヘッダは無視）。課金は既存の `deduct_credits_via_state` をそのまま流用し、非ストリームは残高不足時に402で応答を渡さず、ストリームは生成しながらdrain方式で消費する。未認証アクセスは無料のNemotron固定＋10 req/minのIPレート制限と、OpenAI互換エンドポイントと同一のポリシーを適用している。

## 使い方: 環境変数3つだけ

```bash
export ANTHROPIC_BASE_URL="https://api.teai.io"
export ANTHROPIC_AUTH_TOKEN="te_your_api_key"
export ANTHROPIC_MODEL="moonshotai/kimi-k3"
```

これでClaude Codeを起動すると、内部的にはteai.io経由でOpenRouter提供のKimi K3にリクエストが飛ぶ。`ANTHROPIC_MODEL` を変えれば、カタログ上の他のモデルにもそのまま切り替えられる。

## 動作確認

本番環境に対して、非ストリーム／ストリーム双方のtext応答、tool_useの発行、マルチターンでのtool_result往復を実際のcurlリクエストで確認した。ユニットテストも変換ロジック（リクエスト・レスポンス双方向）に対して9件追加している。

## おわりに

OpenAI互換だけでは届かなかった「Anthropic Messages APIしか話せないクライアント」にも、変換レイヤーを1枚足すことでteai.ioのカタログをそのまま使えるようにした。今後もAnthropic SDKベースのツールが増えれば、このエンドポイントがそのまま活きるはずだ。

フィードバックや質問はコメントでどうぞ。
