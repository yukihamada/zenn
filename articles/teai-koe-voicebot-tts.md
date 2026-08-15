---
title: "LLMの返答を本人声で読み上げるボイスボットを、1つのAPIで作る（teai.io Koe TTS）"
emoji: "🎤"
type: "tech"
topics: ["llm", "tts", "api", "音声合成", "ボイスボット", "音声AI", "python"]
published: true
---

## はじめに

LLMのAPI（ChatGPT、Claude、Geminiなど）はどれも「テキストを返す」ところまで。そこから先、**返答を自然な声でしゃべらせる**には、別の音声合成サービスを契約して認証も料金も別々に管理する必要がある。

[teai.io](https://teai.io) は日本発のLLM APIゲートウェイ。OpenAI互換の1エンドポイントから95以上のモデル（Claude / GPT / Gemini / DeepSeek / Kimi K3など）を呼べる。そして**本人クローン声の音声合成「Koe」も、同じAPIキー・同じOpenAI互換形式で呼べる**。

この記事では、LLMの返答をKoeの本人声で読み上げる最小のボイスボットを、`/v1/audio/speech` に `model:"koe"` を渡すだけで作る手順を解説する。コールセンター自動応答・ナレーション生成・音声エージェントの土台になる。

## 必要なもの

- teai.io の API キー（[無料登録](https://teai.io/register)で1,000クレジット付与）
- Python 3.8+ と OpenAI SDK（`pip install openai`）

## ステップ1：LLMで返答を生成

まずはいつもの `/v1/chat/completions` で返答テキストを作る。モデルは用途で選べる。ここではコストと日本語品質のバランスで Kimi K3 を使う。

```python
from openai import OpenAI

client = OpenAI(
    base_url="https://api.teai.io/v1",
    api_key="teai_...",
)

resp = client.chat.completions.create(
    model="moonshotai/kimi-k3",   # 短縮名 "kimi-k3" でも可
    messages=[{"role": "user", "content": "今日の東京の天気を教えて"}],
)
answer = resp.choices[0].message.content  # → "今日の東京は晴れ、最高気温は…"
```

## ステップ2：Koe本人声で読み上げ

返答テキストを `/v1/audio/speech` に渡し、`model:"koe"` を指定するだけ。機械的な読み上げではなく、**本人クローン声**で日本語を自然に話すMP3が返る。

```python
speech = client.audio.speech.create(
    model="koe",
    input=answer,
    voice="default",   # 省略可。公開声は default / yuki / kentaro / female
)
speech.write_to_file("reply.mp3")
```

curl ならこう。

```bash
curl -X POST https://api.teai.io/v1/audio/speech \
  -H "Authorization: Bearer teai_..." \
  -H "Content-Type: application/json" \
  -d '{"model":"koe","input":"こんにちは、teai.ioです","voice":"default"}' \
  --output reply.mp3
```

`model:"koe"` を指定するのがポイント。OpenAI互換の `/v1/audio/speech` に渡すモデル名を変えるだけでエンジンが切り替わる。`voice` で声を選べる（`default` = 本人声、ほか `yuki` / `kentaro` / `female`）。

## ステップ3：返答→読み上げをループにする

入力（ユーザーの発話テキスト）を受け取るたびに、この2つを回すだけ。最小のボイスボットは下記で完成する。

```python
def reply_voice(user_text: str) -> str:
    resp = client.chat.completions.create(
        model="moonshotai/kimi-k3",
        messages=[
            {"role": "system", "content": "あなたは店舗の電話応対係です。丁寧な日本語で簡潔に答えてください。"},
            {"role": "user", "content": user_text},
        ],
    )
    answer = resp.choices[0].message.content
    speech = client.audio.speech.create(model="koe", input=answer)
    speech.write_to_file("reply.mp3")
    return answer
```

## 何が嬉しいのか

海外のAPIゲートウェイには「日本語の本人声TTS」が無い。LLMと音声を**1つのAPIキー・1つの請求**で完結できるのがteai.ioの差別化ポイントだ。

| 用途 | 組み合わせ |
|---|---|
| コールセンター自動応答 | LLM（応対文生成）→ Koe（本人声で読み上げ） |
| ナレーション生成 | 原稿をKoeで一括読み上げ（ブログ・動画・紙芝居の音声） |
| 音声エージェント | LLM + Koe をCLIエージェントや自社アプリに配線 |
| 社内アナウンス | 定型文をKoeで合成して通知・放送に |

## 料金

Koe TTS は**約1クレジット/100文字**（1文字あたり約0.01クレジット）と安価。無料登録でもらえる1,000クレジットで、数千文字分は追加コストなしで試せる。LLM側の利用もプロバイダー価格そのまま＋マージン5%。最新の料金は[公式の料金ページ](https://teai.io/pricing)を参照。

## 制約（正直に）

- **音声入力（STT）は未提供**。ボイスボットの「聞き取り」部分は、お手持ちの Whisper / Google STT / ブラウザの Web Speech API などと組み合わせてほしい。話す側（LLM応答＋Koe読み上げ）はこの記事の通りすぐ使える。
- **1回の入力は1〜4096文字**。長文は分割して呼ぶ。
- 公開されている声は `default`（本人声）/ `yuki` / `kentaro` / `female`。自分の声を登録したい場合は [koe.live](https://koe.live) から。

## まとめ

LLMの返答を本人クローン声でしゃべらせるボイスボットが、`model:"koe"` を1つ指定するだけで作れる。音声エージェントやコールセンター用途を考えているなら、まず無料枠のクレジットで「LLMの返答を声にする」体験から始めてみてほしい。

- teai.io 登録: https://teai.io/register
- 料金: https://teai.io/pricing
- 関連記事: [Claude CodeからKimi K3を直結で使う](https://zenn.dev/yukihamada/articles/teai-claude-code-kimi-k3)
- アーキテクチャ: [日本発LLM API Gateway teai.ioのアーキテクチャを全公開](https://zenn.dev/yukihamada/articles/teai-llm-api-gateway-architecture)
