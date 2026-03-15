---
title: "Koe v2.6 開発記録 — カスタム辞書・Raycast拡張・ハルシネーション対策まで一気にやった話"
emoji: "🎙"
type: "tech"
topics: ["whisper", "swift", "macos", "音声認識", "llm"]
published: true
---

Mac の音声入力アプリ [Koe](https://koe.elio.love/) を今日一日でガッと改善した記録。機能追加から設定UI刷新、Whisperのハルシネーション対策まで。

---

## 今日やったこと一覧

| 作業 | 内容 |
|------|------|
| Whisper translate フラグ | LLM不要でWhisperが直接英訳 |
| カスタム辞書 | 固有名詞・専門用語をWhisperのpromptに注入 |
| Raycast プラグイン | `koe://` URLスキーム経由で音声入力を制御 |
| ブラウザ拡張 | Chrome/Safari対応のポップアップUI |
| 設定UI刷新 | best_of・Temperature・エントロピー閾値など全パラメータを公開 |
| ハルシネーション対策 | 「と言われてるんだけどね」問題を3段構えで解決 |

---

## Whisper native translate

これが地味に大きかった。Whisperには元々`translate`フラグがあって、これをONにすると**LLMを一切使わずに**音声を直接英語に変換してくれる。

従来の翻訳フロー:
```
音声 → Whisper(日本語) → LLM(翻訳) → 英語テキスト
```

native translateのフロー:
```
音声 → Whisper(translate=true) → 英語テキスト
```

速度が全然違う。LLMのAPI呼び出しがなくなるので、ほぼWhisperの認識時間だけで英訳完了。

実装はCブリッジに`bool translate`を追加して、SwiftからそのままWhisperに渡すだけ:

```c
// whisper_bridge.c
params.translate = translate;
```

```swift
// SpeechEngine.swift
let useNativeTranslate = AppSettings.shared.llmMode == .translate
WhisperContext.shared.transcribe(url: url, language: lang, prompt: prompt,
                                 translate: useNativeTranslate) { text in
```

---

## カスタム辞書

Whisperは汎用モデルなので、`OpenClaw`や`yukihamada`みたいな固有名詞は認識ミスしやすい。WhisperのAPIには`initial_prompt`という引数があって、**これに単語を渡しておくと認識精度が上がる**。

設定画面から単語を登録しておくと、認識のたびに`initial_prompt`に自動追加される仕組み:

```swift
// ContextCollector.swift
let vocab = settings.customVocabulary.filter { !$0.isEmpty }
if !vocab.isEmpty {
    parts.append(vocab.joined(separator: "、"))
}
```

UIはタグ入力UIで登録・削除できるようにした。

```
[OpenClaw ×] [yukihamada ×] [Koe ×]
[単語を追加…] [追加]
```

---

## Raycast プラグイン

Koeには`koe://transcribe`などのURLスキームが実装済みなので、Raycastプラグインは超シンプル:

```typescript
// start-recording.ts
export default async function Command() {
  await open("koe://transcribe");
  await showHUD("🎙 Koe: Voice Input Started");
}
```

コマンド一覧:
- **Start Voice Input** — `koe://transcribe`
- **Start Voice Translation** — `koe://translate`
- **Stop Recording** — `koe://stop`
- **Open Settings** — `koe://settings`

`koe://stop`と`koe://settings`は今回新しく追加したエンドポイント。

---

## ブラウザ拡張 (Chrome/Safari)

同じくURLスキーム経由でブラウザのツールバーからKoeを操作できるようにした。Manifest V3準拠:

```json
{
  "manifest_version": 3,
  "action": {
    "default_popup": "popup.html"
  }
}
```

ポップアップUIで4つのアクションを選べる。SafariはXcodeの`safari-web-extension-converter`でそのまま変換可能。

---

## 設定UI刷新 — best_of を公開

Whisperの`best_of`パラメータは「デコーダがいくつの候補を生成して一番良いものを選ぶか」を決める値。

| best_of | 速度 | 精度 |
|---------|------|------|
| 1 | ⚡ 最速 (3-4x) | 標準 |
| 3 | 普通 | 良い |
| 5 | 遅い | 最高 |

CoreML(Apple Neural Engine)でエンコーダが高速化されているので、デコーダのbest_ofを1にしても十分な精度が出る。デフォルトを1にした。

今まで隠れていたWhisperの全パラメータを設定画面から変更できるようにした:

- **best_of** (デコーダ候補数)
- **Temperature** (認識のゆらぎ)
- **Temperature増分** (認識失敗時の再試行設定)
- **エントロピー閾値** (不確かな認識の棄却)
- **話し終わりの待ち時間** (0.8s〜5.0s)
- **Beam Search** / **文脈を引き継ぐ**

---

## ハルシネーション対策

今日一番困ったのがこれ。何も言っていないのに:

> 「と言われてるんだけどね。」

が出力される。これはWhisperの**ハルシネーション(幻覚)**で、音量が低い・無音に近いときにトレーニングデータ(YouTube動画)で多く見たフレーズを補完してしまう現象。

**3段構えで対策した:**

### 1. 音声検出の閾値を引き上げ

```swift
// Before
AudioDSP.hasVoice(samples, threshold: 0.003, minVoiceFrames: 3)

// After
AudioDSP.hasVoice(samples, threshold: 0.006, minVoiceFrames: 5)
```

より明確な音声があるときだけWhisperに渡すようにした。

### 2. no_speech_thold を引き上げ

```c
// Before: 0.6
// After: 0.7
params.no_speech_thold = 0.7;
```

Whisper内部の「音声なし」判定基準を厳しくして、無音時の幻覚を抑制。

### 3. ハルシネーションフィルタ

完全一致・部分一致のブロックリストで事後フィルタ:

```swift
static func filterHallucination(_ text: String, rms: Float) -> String {
    let exactBlocklist: Set<String> = [
        "と言われてるんだけどね",
        "と言われてるんだけどね。",
        "ご視聴ありがとうございました",
        "チャンネル登録よろしくお願いします",
        // ... 20種類
    ]
    if exactBlocklist.contains(text) { return "" }

    // 低音量時は部分一致でも除去
    if rms < 0.01 {
        if text.contains("と言われてるんだけど") { return "" }
        // ...
    }
    return text
}
```

---

## 振り返り

ハルシネーションは「no_speech_tholdを上げれば全部解決」と思っていたが、実際にはWhisperが内部でVADを通した後にもハルシネーションが発生するので、事後フィルタも必要だった。

カスタム辞書は実装が簡単なわりに効果が高い機能で、もっと早くやれば良かった。`initial_prompt`は本来「前の発話テキスト」として設計されているが、キーワードを渡しても認識精度向上に使えるのはWhisperの面白い特性。

---

## Koeについて

macOSで動くローカル音声入力アプリ。whisper.cpp + Metal GPU + CoreML(Apple Neural Engine)で高速認識。音声データは一切クラウドへ送信しない。

- サイト: https://koe.elio.love/
- GitHub: https://github.com/yukihamada/Koe-swift
- インストール: `Koe.pkg` をダブルクリックするだけ (Apple公証済み)
