---
title: "Koe v2.6 開発記録 — カスタム辞書・Raycast拡張・ハルシネーション対策を一日で全部やった"
emoji: "🎙"
type: "tech"
topics: ["whisper", "swift", "macos", "音声認識", "raycast"]
published: true
---

**TL;DR**: Mac音声入力アプリ [Koe](https://koe.elio.love) に「カスタム辞書」「Whisper native translate」「Raycastプラグイン」「ブラウザ拡張」を一日で実装した。ついでにWhisperが「と言われてるんだけどね。」と勝手に喋る問題を3段構えで直した。

---

## 今日の積み上げ

[前回の記事](https://zenn.dev/yukihamada/articles/koe-coreml-whisper-5x-faster)でCoreMLによる5倍高速化まで終わっていた。今日は「もっと賢くする」フェーズ。

---

## 1. Whisper native translate

まず一番シンプルなやつ。翻訳モードのLLM依存をなくした。

Koeには「翻訳モード」があって、従来は:

```
音声 → Whisper (日本語) → LLM API (翻訳) → 英語テキスト
```

という流れだった。LLM APIを叩くのでネット必須・レイテンシあり。

Whisperには元々`translate`フラグがある。これをONにするだけで直接英語を出力してくれる:

```
音声 → Whisper (translate=true) → 英語テキスト
```

速い。LLMのAPI呼び出しがなくなるので、ほぼWhisper単体の処理時間で英訳が完了する。

実装はCブリッジに`bool translate`を追加するだけ:

```c
// whisper_bridge.c
params.translate = translate;
```

```swift
// SpeechEngine.swift — 翻訳モード時にフラグを立てる
let useNativeTranslate = AppSettings.shared.llmMode == .translate
WhisperContext.shared.transcribe(url: url, language: lang, prompt: prompt,
                                 translate: useNativeTranslate) { text in
```

---

## 2. カスタム辞書

Whisperは汎用モデルなので固有名詞に弱い。`OpenClaw`は「オープンクロー」になるし、`yukihamada`は「幸田ましろ」とか謎の変換になる。

Whisperに`initial_prompt`という引数がある。本来は「前の会話の続き」として使う機能だが、**単語リストを渡しても認識精度向上に使える**。

設定画面から単語を登録しておくと認識のたびに自動でpromptに注入する:

```swift
// ContextCollector.swift
let vocab = settings.customVocabulary.filter { !$0.isEmpty }
if !vocab.isEmpty {
    parts.append(vocab.joined(separator: "、"))
}
```

UIはタグ入力。追加・削除できる:

```
[OpenClaw ×] [yukihamada ×] [Koe ×]
[ 単語を追加…            ] [追加]
```

`initial_prompt`は長すぎると逆に精度が落ちる（Whisperがプロンプトを「前の発話」として扱うため）。100文字以内に制限している。

---

## 3. Raycast プラグイン

KoeにはURLスキームが実装済みなので、Raycastプラグインは薄いラッパーだけで済む:

```typescript
// start-recording.ts
export default async function Command() {
  await open("koe://transcribe");
  await showHUD("🎙 Koe: Voice Input Started");
}
```

コマンド一覧:

| コマンド | URLスキーム |
|---------|------------|
| Start Voice Input | `koe://transcribe` |
| Start Voice Translation | `koe://translate` |
| Stop Recording | `koe://stop` |
| Open Settings | `koe://settings` |

`koe://stop`と`koe://settings`は今回新しく追加したエンドポイント。AppDelegateに2行追加するだけ:

```swift
case "stop":
    guard isRecording else { return }
    DispatchQueue.main.async { self.stopAndRecognize() }
case "settings":
    DispatchQueue.main.async { self.openSettings() }
```

---

## 4. ブラウザ拡張 (Chrome/Safari)

同じくURLスキーム経由。Manifest V3:

```json
{
  "manifest_version": 3,
  "action": { "default_popup": "popup.html" }
}
```

ポップアップから音声入力・翻訳・停止・設定を操作できる。Safariは`xcrun safari-web-extension-converter`でそのまま変換できる。

---

## 5. 設定UIの刷新 — best_of を公開

Whisperの`best_of`は「デコーダが何個の候補を生成して最良を選ぶか」を決める値。

| best_of | 速度 | 精度 |
|---------|------|------|
| 1 | ⚡ 最速 (3-4x) | 標準 |
| 3 | 普通 | 良い |
| 5 | 遅い | 最高 |

CoreMLでエンコーダが高速化されているので、デコーダを`best_of=1`にしても十分な精度が出る。デフォルトを5→1に変更した（前バージョンv2.5.1の変更）。

今まで内部に隠れていたWhisperパラメータを全部設定画面から変更できるようにした:

- **best_of** — 候補数 (1/2/3/5)
- **Temperature** — 認識のゆらぎ (0〜0.6)
- **Temperature増分** — 認識失敗時の再試行設定 (0〜0.4)
- **エントロピー閾値** — 不確かな認識の棄却 (2.0〜3.0)
- **話し終わりの待ち時間** — 0.8s〜5.0s

---

## 6. ハルシネーション対策 — 「と言われてるんだけどね」問題

今日一番困ったのがこれ。何も言っていないのに:

> 「と言われてるんだけどね。」

が出力される。

これはWhisperの**ハルシネーション**（幻覚）。音量が低い・無音に近いとき、トレーニングデータ（YouTube動画）で頻出したフレーズを補完してしまう既知の問題。日本語は特にYouTubeっぽいフレーズが出やすい。

3段構えで対策した。

### 対策1: 音声検出の閾値を引き上げ

```swift
// Before: 小さな音も通してしまう
AudioDSP.hasVoice(samples, threshold: 0.003, minVoiceFrames: 3)

// After: より明確な音声だけWhisperに渡す
AudioDSP.hasVoice(samples, threshold: 0.006, minVoiceFrames: 5)
```

### 対策2: no_speech_thold を引き上げ

```c
// Before: 0.6 — 低すぎてWhisperが無音を「音声あり」と誤判定
// After: 0.7
params.no_speech_thold = 0.7;
```

Whisper内部の「音声なし」判定を厳しくした。

### 対策3: 事後フィルタ

上2つで防げなかったケース用のブロックリスト。完全一致と、低音量時の部分一致:

```swift
static func filterHallucination(_ text: String, rms: Float) -> String {
    let exactBlocklist: Set<String> = [
        "と言われてるんだけどね",
        "と言われてるんだけどね。",
        "ご視聴ありがとうございました",
        "チャンネル登録よろしくお願いします",
        "ありがとうございました",
        // ... 計20種類
    ]

    if exactBlocklist.contains(text) { return "" }

    // 低音量時 (rms < 0.01) は部分一致でも除去
    if rms < 0.01 {
        if text.contains("と言われてるんだけど") { return "" }
        if text.contains("チャンネル登録") { return "" }
    }
    return text
}
```

ログに `hallucination filtered` が出たら正常動作。

---

## まとめ

- **Whisper native translate**: LLM APIを使わず翻訳完了。Cブリッジに`bool translate`追加するだけ
- **カスタム辞書**: `initial_prompt`に単語を注入。100文字以内制限が重要
- **Raycast/ブラウザ拡張**: URLスキームがあれば薄いラッパーで作れる
- **ハルシネーション**: 音声検出閾値 + no_speech_thold + 事後フィルタの3段構え。1つだけでは足りない

ハルシネーションは「no_speech_tholdを上げれば全部解決」と思っていたが、Whisperが内部でVADを通した後にも発生するので事後フィルタが必要だった。WhisperのVADと独自VADで二重チェックする設計が正解。

---

## Koe v2.6.0

今日の変更は [v2.6.0](https://github.com/yukihamada/Koe-swift/releases/tag/v2.6.0) としてリリース済み。

```bash
# インストール
curl -fsSL https://koe.elio.love/install.sh | bash
```

- サイト: https://koe.elio.love
- GitHub: https://github.com/yukihamada/Koe-swift
- PKG直接DL: [Koe.pkg](https://github.com/yukihamada/Koe-swift/releases/latest/download/Koe.pkg) (Apple公証済み、ダブルクリックでインストール)
