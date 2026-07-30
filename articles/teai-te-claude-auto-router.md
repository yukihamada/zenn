---
title: "te claude 一発でClaude Codeの裏側をKimi K3に — teai/auto で安⇔賢を自動切替"
emoji: "🔀"
type: "tech"
topics: ["claudecode", "llm", "kimi", "rust", "api"]
published: true
---

## TL;DR

- `te claude` — Claude Code を [teai.io](https://teai.io) 経由で起動するランチャーを作った。環境変数の手動設定なしで、裏側のモデルを Kimi K3 などカタログ上の任意モデルに差し替えられる
- `model: "teai/auto"` — リクエスト内容を見て「短い質問→安いモデル / コードや長文→Kimi K3」を**リクエスト単位で自動選択**するルーターを実装した。課金とレスポンスの `model` フィールドは解決後の実モデルで返す
- モデル系統ごと落ちた場合の**クロスモデル自動フォールバック**も入れた（`x-teai-fallback-model` ヘッダで明示）

前回記事（[Anthropic Messages API互換を自作した話](https://zenn.dev/yukihamada/articles/teai-claude-code-kimi-k3)）の続編で、今回は「使う側の体験」を仕上げた話。

## 1. `te claude` — 起動コマンド1つで裏側を差し替える

teai.io には `te`（Sente）という CLI がある。インストールは1行:

```bash
curl -fsSL https://teai.io/te | sh
te login
```

今回 `te claude` サブコマンドを追加した:

```bash
te claude                        # 保存済みのデフォルトモデルで Claude Code 起動
te claude moonshotai/kimi-k3     # このセッションだけ K3 指定
```

内部でやっているのは `ANTHROPIC_BASE_URL` / `ANTHROPIC_AUTH_TOKEN` / `ANTHROPIC_MODEL` をセットして `claude` を exec するだけ。手で書くならこう:

```bash
export ANTHROPIC_BASE_URL="https://api.teai.io"
export ANTHROPIC_AUTH_TOKEN="te_your_api_key"
export ANTHROPIC_MODEL="teai/auto"   # ← 後述の自動切替
```

モデルの永続切替も CLI から:

```bash
te model                       # 一覧+現在のデフォルト表示
te model moonshotai/kimi-k3    # デフォルト変更（設定ファイルに保存）
te auto / te fast / te max     # その回だけ 自動/最安/K3
```

## 2. `model: "teai/auto"` — リクエスト単位の自動ルーティング

「全部 K3 に投げると高い。全部安いモデルだと重いタスクで失敗する」— なら**リクエストごとに振り分ければいい**、というのが `teai/auto`。

`/v1/chat/completions`（OpenAI互換）と `/v1/messages`（Anthropic互換）の両方で、`model` に `teai/auto` を指定すると、サーバ側のヒューリスティックが実モデルを選ぶ:

| リクエストの特徴 | 選択されるモデル |
|---|---|
| ツール呼び出しあり / コードブロック含む / 長文 / 実装・デバッグ系キーワード | `moonshotai/kimi-k3` |
| 短い質問・雑談 | `deepseek-chat` |
| その中間 | `meta-llama/llama-4-maverick` |

設計で守ったのは2点:

1. **課金とレスポンスの `model` フィールドは、解決後に実際に使ったモデル**で返す。「auto と言いながら何が動いたか分からない」を作らない
2. 判定理由をレスポンスヘッダ `x-teai-auto-reason` で返す。クライアント側から「なぜこのモデルになったか」を確認できる

本番での実測（公開エンドポイントに対する E2E）:

```
model:"teai/auto" + "こんにちは"（短文）
  → 応答の model: deepseek-chat

model:"teai/auto" + コードブロック+デバッグ依頼
  → 応答の model: moonshotai/kimi-k3
```

Claude Code の `ANTHROPIC_MODEL` に `teai/auto` を入れておくと、**エージェントの1ターンごとに安⇔賢が勝手に切り替わる**。体感は「賢いときは賢い、財布は軽くならない」。

## 3. クロスモデル自動フォールバック

従来の teai.io のフェイルオーバーは「同一モデルを提供する複数プロバイダ間」だけだった。今回、要求モデルの全プロバイダが落ちている／サーキットブレーカーが開いている場合に、**別モデルへ1段だけ代替**するチェーンを追加した（例: Kimi K3 → GLM系 → GPT系 → Llama）。

- 代替が発動した場合はレスポンスヘッダ `x-teai-fallback-model` で必ず明示
- 環境変数 `NANOBOT_NO_FALLBACK=1` で無効化可能（「黙って別モデルに変わるのは嫌だ」派のため）

エージェント用途だと「上流モデルの一時障害でセッションごと死ぬ」のが一番痛いので、明示付きの1段フォールバックが実用のバランスだと考えている。

## 4. スラッシュコマンド `/model`

チャットUI側にも `/model` を追加した。`/model` で主要モデルの一覧（単価付き）と現在のデフォルト、`/model deepseek-chat` で自分のデフォルトモデルを永続変更できる。

実装中に、ユーザーデフォルト設定の永続化が旧 DynamoDB 専用実装のままで**本番（libSQL）では常に無効になっていた**バグを見つけて直した。こういうのは机上レビューでは出ない。E2E で「設定→再読込→反映」のラウンドトリップを回して初めて気づけた。

## 実装メモ

- 対象リポジトリは Rust (axum) + Fly.io。ルーター本体は `provider/auto_router.rs` として独立させ、選択ロジックはユニットテストで固定（20ケース）
- 面白かった落とし穴: フォールバック無効化の env var をテストが並列に触って CI が間欠的に落ちた。env var を触るテストは共有 Mutex で直列化するのが Rust の定石
- カタログと課金レートは `PRICING_TABLE` が単一ソースなので、`teai/auto` も表示用エントリとしてそこに足した（課金は解決先モデルのレート）

## おわりに

「モデルを選ぶ」こと自体をユーザーの仕事から外していくのが方向性。`te claude` で入口を1コマンドに、`teai/auto` で選択を自動に、フォールバックで障害時も止まらないように、の3点セットでかなり近づいた。

フィードバックはコメントか [teai.io](https://teai.io) からどうぞ。
