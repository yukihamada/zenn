---
title: "Claude Codeの従量課金が高すぎて、自分でLLMゲートウェイを作った話（teai.io と sente）"
emoji: "💸"
type: "tech"
topics: ["claudecode", "llm", "opencode", "個人開発", "api"]
published: true
---

:::message
筆者は teai.io を運営する株式会社イネブラの代表です。当事者の紹介記事として読んでください。価格はすべて 2026-09-02 時点の公開情報・公開 API の実測値です。
:::

## 3行で

- Claude Code はそのまま、**接続先だけ**を環境変数3つで teai.io に向けると、裏のモデルを作業ごとに選べる
- Claude Sonnet 5 / Opus 5 / Fable 5.1 も公開建値より 3% 安く、GLM-5.2 や DeepSeek V4 Flash に振れば **同じ作業が 1/6〜1/14 の単価**になる
- 円建て・インボイス対応。OpenCode をフォークした国産エージェント **sente** も付いてくる

## きっかけは、毎月の請求

Claude Code は本当によくできています。ただ、1日中エージェントを回していると従量課金の請求がかさむ。定額プランも Pro が月 $20、Max は月 $100 から（[claude.com/pricing](https://claude.com/pricing)）で、円で見ると重い。

API 単価は Claude Sonnet 5 が入力 $2 / 出力 $10（100万トークンあたり）、Opus 5 が $5 / $25、Fable 5.1 が $10 / $50（[公式 pricing](https://platform.claude.com/docs/en/about-claude/pricing)）。エージェントはツール呼び出しのたびにコンテキストを送り直すので、体感より入力トークンが積み上がります。

一方で、エージェントの作業の大半は「ファイルを読む」「テストを回す」「差分を書く」の反復です。全部に最上位モデルを使う必要はない。**Claude Code の使い勝手はそのまま、裏のモデルだけ用途に応じて替えたい**。それだけのために作ったのが [teai.io](https://teai.io) です。

## 何を作ったか

teai.io は東京リージョンで動く LLM API ゲートウェイです。

- **Anthropic Messages API 互換の `/v1/messages`** を実装。Claude Code や Anthropic SDK から変換プロキシなしで直結できる
- OpenAI 互換の `/v1/chat/completions` もあるので、既存コードは `base_url` を変えるだけ
- 1 つの API キーで 380+ モデル（Claude・GPT・Gemini・GLM・DeepSeek・Qwen・Kimi など）
- 値付けは各モデルの公開建値から 3% 引きで自動更新。売価は [`teai.io/api/v1/pricing`](https://teai.io/api/v1/pricing) で誰でも見られる
- 円建て決済・適格請求書（インボイス）対応

## Claude Code の接続先を変える（環境変数3つ）

```bash
export ANTHROPIC_BASE_URL=https://api.teai.io
export ANTHROPIC_AUTH_TOKEN=te_xxxx        # teai.io/register で発行
export ANTHROPIC_MODEL=z-ai/glm-5.2        # 使いたいモデルID
claude
```

これだけです。`ANTHROPIC_MODEL` を `anthropic/claude-sonnet-5` にすれば Claude のまま、`anthropic/claude-fable-5.1` にすれば Fable、`deepseek/deepseek-v4-flash` にすれば DeepSeek に切り替わります。Claude Code 側の設定やスラッシュコマンドはそのまま使えます。

この記事を書く前に、上の設定で GLM-5.2 と Fable 5.1 の両方を `/v1/messages` 経由で実際に叩き、正常に応答が返ることを確認しています。

## 単価の実測（2026-09-02・teai.io の公開価格 API）

100万トークンあたりの USD 換算で、Anthropic 直契約と並べたものが下の表です。

| モデル | 入力 | 出力 | 備考 |
|---|---|---|---|
| Claude Sonnet 5（Anthropic 直） | $2.00 | $10.00 | 公式 |
| Claude Sonnet 5（teai.io 経由） | $1.94 | $9.70 | 建値の 3% 引き |
| Claude Opus 5（teai.io 経由） | $4.85 | $24.25 | 同上 |
| Claude Fable 5.1（teai.io 経由） | $9.70 | $48.50 | 同上。最上位・1M コンテキスト |
| GLM-5.2 | $0.32 | $1.02 | Sonnet 5 の約 1/6〜1/10 |
| DeepSeek V4 Flash | $0.23 | $0.69 | 約 1/8〜1/14 |
| MiniMax M3 | $0.29 | $1.16 | |
| Qwen3.7 Flash | $0.03 | $0.13 | 軽い作業向け |
| Kimi K3 | $2.91 | $14.55 | **安くない**。Sonnet 5 より高い |

### 1日回すと、いくら違うか

Claude Code を1日中使ったときの目安として、**1日に入力 500万トークン・出力 50万トークン**を消費する、と仮定して建値で計算した表です（キャッシュなし。あくまで仮定の量です）。

| 使うモデル | 1日 | 月20営業日 |
|---|---|---|
| Claude Sonnet 5（Anthropic 直） | $15.0 | $300 |
| Claude Sonnet 5（teai.io） | $14.6 | $291 |
| Claude Fable 5.1（teai.io） | $72.8 | $1,455 |
| Kimi K3（teai.io） | $21.8 | $436 |
| GLM-5.2（teai.io） | $2.1 | $43 |
| MiniMax M3（teai.io） | $2.0 | $41 |
| DeepSeek V4 Flash（teai.io） | $1.5 | $30 |

正直に書いておくと、話題の Kimi K3 は teai.io 経由でも Sonnet 5 より高いです（上流の OpenRouter・DeepInfra の建値がそもそも $3 / $15 前後）。「安くする」目的なら GLM-5.2・DeepSeek V4 Flash・MiniMax M3 が現実的な選択肢で、リファクタやテスト生成のような定型作業はこのクラスで十分回ります。設計判断やデバッグの山場だけ Claude に戻す、という使い分けが一番効きました。

なお Claude Code は Anthropic 直だとプロンプトキャッシュが効くので、実際の請求は上の表より下がります。teai.io 経由でのキャッシュ割引の適用は本稿では検証していません。この表は「建値で見たときの桁の違い」を掴むためのものです。

上流は OpenRouter や DeepInfra などを経由しており、どのモデルがどの経路を通るかは価格 API の `provider` 欄で開示しています。国内完結の経路はまだありません。

## 国産エージェント sente（先手）

Claude Code に挿すだけでも良いのですが、teai.io には自前のコーディングエージェント **sente** も付いています。

- 本体は **OpenCode（MIT ライセンスの OSS）のフォーク**です（[github.com/yukihamada/opencode](https://github.com/yukihamada/opencode)）。上流の更新を取り込みつつ、Sente へのリブランド、起動の高速化（使わないプロバイダのカタログ構築をスキップ）、セッション DB の差分読み込み、ホーム画面やヒント表示、teai.io 向けの既定値などを加えています
- インストーラの `te` が teai.io から設定を取得し、日本語の作業ルール・メモリ・声の設定を注入する。「このバグ直して、テストも書いて」がそのまま通る
- `te talk` で声で指示しながら開発できる（日本語音声対話）
- teai.io の全モデルを同じ CLI から切り替えられる
- 派生として、C 言語でリライトした軽量版 [sente-c](https://github.com/yukihamada/sente-c)（alpha）と、ブラウザから使う [sente-cloud](https://github.com/yukihamada/sente-cloud) も公開しています

導入は 1 行です。GitHub Releases のバイナリを取得して `te` コマンドを用意します。

```bash
curl -fsSL https://teai.io/te | sh
te run "このリポジトリを説明して"
```

詳しくは [teai.io/sente](https://teai.io/sente) にまとめています。

## こんな人に向いています

- Claude Code の請求を見て「定型作業まで最上位モデルに払うのは違う」と感じている
- ドル建てのカード明細を円換算して経費計上するのに疲れた。請求書払い・インボイスが欲しい
- モデルを乗り換えるたびに SDK やコードを書き換えたくない
- 日本語で指示して日本語で返ってくるエージェントを、ターミナルで使いたい

逆に、社外にデータを一切出せない要件（国内完結・専有環境）には現時点では向きません。

## 無料で試す

登録はメールアドレスのみ、カード不要です。登録すると 100 クレジットが付き、軽量モデルなら千回以上、Claude Sonnet 5 クラスでも十回程度は試せます。

- 登録: [teai.io/register](https://teai.io/register)
- 料金: [teai.io/pricing](https://teai.io/pricing)
- 価格 API: `https://teai.io/api/v1/pricing`

## おわりに

「高いから安いのを使え」という話ではなく、**作業の種類ごとに払う値段を選べるようにしたい**というのが動機でした。Claude Code という道具は変えずに、裏側だけ選べる。同じ悩みを持つ人の月末が少し軽くなれば嬉しいです。

フィードバックや「このモデルも載せて」は X（[@yukihamada](https://x.com/yukihamada)）まで。
