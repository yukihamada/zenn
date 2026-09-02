---
title: "Claude Codeの従量課金が高すぎて、自分でLLMゲートウェイを作った話（teai.io / sente）"
emoji: "💸"
type: "tech"
topics: ["claudecode", "llm", "opencode", "個人開発", "api"]
published: true
---

## きっかけは、毎月の請求

Claude Code は本当によくできています。ただ、1日中エージェントを回していると従量課金の請求がかさむ。定額プランも Pro が月 $20、Max は月 $100 から（[claude.com/pricing](https://claude.com/pricing)、2026-09-02 時点）で、円で見ると重い。

API 単価は Claude Sonnet 5 が入力 $2 / 出力 $10（100万トークンあたり）、Opus 5 が $5 / $25（[公式 pricing](https://platform.claude.com/docs/en/about-claude/pricing)）。エージェントはツール呼び出しのたびにコンテキストを送り直すので、体感より入力トークンが積み上がります。

「Claude Code の使い勝手はそのまま、裏のモデルだけ用途に応じて安いものに替えたい」。それだけのために、自分でゲートウェイを作りました。それが [teai.io](https://teai.io) です。

## 何を作ったか

teai.io は東京リージョンで動く LLM API ゲートウェイです。

- **Anthropic Messages API 互換の `/v1/messages`** を実装。Claude Code や Anthropic SDK から変換プロキシなしで直結できる
- OpenAI 互換の `/v1/chat/completions` もあるので、既存コードは `base_url` を変えるだけ
- 1 つの API キーで 380+ モデル（Claude・GPT・Gemini・GLM・DeepSeek・Qwen・Kimi など）
- 円建て決済・適格請求書（インボイス）対応
- 運営は株式会社イネブラ（東京）。筆者はその代表なので、この記事は当事者の紹介記事です

## Claude Code の接続先を変える（環境変数3つ）

```bash
export ANTHROPIC_BASE_URL=https://api.teai.io
export ANTHROPIC_AUTH_TOKEN=te_xxxx        # teai.io/register で発行
export ANTHROPIC_MODEL=z-ai/glm-5.2        # 使いたいモデルID
claude
```

これだけです。`ANTHROPIC_MODEL` を `anthropic/claude-sonnet-5` にすれば Claude のまま、`deepseek/deepseek-v4-flash` にすれば DeepSeek に切り替わります。Claude Code 側の設定やスラッシュコマンドはそのまま使えます。

## 単価の実測（2026-09-02・teai.io の公開価格 API）

teai.io の売価は `https://teai.io/api/v1/pricing` で誰でも確認できます。100万トークンあたりの USD 換算で、Anthropic 直契約と並べたものが下の表です。

| モデル | 入力 | 出力 | 備考 |
|---|---|---|---|
| Claude Sonnet 5（Anthropic 直） | $2.00 | $10.00 | 公式 |
| Claude Sonnet 5（teai.io 経由） | $1.94 | $9.70 | 公開建値の 3% 引きで自動値付け |
| Claude Opus 5（teai.io 経由） | $4.85 | $24.25 | 同上 |
| GLM-5.2 | $0.32 | $1.02 | Sonnet 5 の約 1/6〜1/10 |
| DeepSeek V4 Flash | $0.23 | $0.69 | 約 1/8〜1/14 |
| MiniMax M3 | $0.29 | $1.16 | |
| Qwen3.7 Flash | $0.03 | $0.13 | 軽い作業向け |
| Kimi K3 | $2.91 | $14.55 | **安くない**。Sonnet 5 より高い |

正直に書いておくと、話題の Kimi K3 は teai.io 経由でも Sonnet 5 より高いです。「安くする」目的なら GLM-5.2・DeepSeek V4 Flash・MiniMax M3 あたりが現実的な選択肢で、リファクタやテスト生成のような定型作業はこのクラスで十分回ります。難しい設計判断だけ Claude に戻す、という使い分けが一番効きました。

上流は OpenRouter や DeepInfra などを経由しており、どのモデルがどの経路を通るかは価格 API の `provider` 欄で開示しています。国内完結の経路はまだありません。

## 国産エージェント sente（先手）

Claude Code に挿すだけでも良いのですが、teai.io には自前のコーディングエージェント **sente** も付いています。

- エージェント本体は OpenCode（MIT ライセンスの OSS）をベースにしています。上流をフォークせず、teai.io 向けの設定とツールを重ねた薄いランチャーなので、OpenCode 本体の進化にそのまま乗れます
- 日本語の指示がそのまま通る。「このバグ直して、テストも書いて」で調査・修正・テストまで進む
- `te talk` で声で指示しながら開発できる（日本語音声対話）
- teai.io の全モデルを同じ CLI から切り替えられる

導入は 1 行です。

```bash
curl -fsSL https://teai.io/te | sh
te run "このリポジトリを説明して"
```

詳しくは [teai.io/sente](https://teai.io/sente) にまとめています。

## 無料で試す

登録はメールアドレスのみ、カード不要です。登録後にクーポン `WELCOME` を適用すると 100 クレジットが付きます（1 アカウント 1 回）。軽量モデルなら千回以上、Claude Sonnet 5 クラスでも十回程度は試せます。

- 登録: [teai.io/register](https://teai.io/register)
- 料金: [teai.io/pricing](https://teai.io/pricing)
- 価格 API: `https://teai.io/api/v1/pricing`

## おわりに

「高いから安いのを使え」という話ではなく、**作業の種類ごとに払う値段を選べるようにしたい**というのが動機でした。Claude Code という道具は変えずに、裏側だけ選べる。同じ悩みを持つ人の月末が少し軽くなれば嬉しいです。

フィードバックや「このモデルも載せて」は X（[@yukihamada](https://x.com/yukihamada)）まで。
