---
title: "Koe v2.7 + iOS v1.1.0 — WidgetKit・Dynamic Island・iPhoneとMacのP2P連携を全部実装した"
emoji: "🎙"
type: "tech"
topics: ["swift", "widgetkit", "activitykit", "appintents", "multipeerconnectivity"]
published: true
---

**TL;DR**: Mac音声入力アプリ [Koe](https://koe.elio.love) をMac v2.7.0・iOS v1.1.0に大型アップデートした。WidgetKit・Siri Shortcuts・Dynamic Island・iPhoneとMacのローカルP2P連携・話者分離・メニューバー履歴——合計7機能を一気に実装した記録。

---

## はじめに

**Koe（声）** は、ぼくが個人開発しているMac・iOS向けの音声入力アプリだ。Macではwhisper.cppをローカルで動かして文字起こしをするため、音声データがクラウドに出ることはない。iPhoneではApple純正の`SFSpeechRecognizer`を使い、こちらもできる限りオンデバイスで処理する。「速い・安全・ネット不要」を軸に作ってきたアプリで、今回はMac版をv2.7.0に、iOS版を新たにv1.1.0へアップデートした。

| プラットフォーム | バージョン | 主な追加機能 |
|---|---|---|
| Mac | v2.7.0 | iPhone連携・メニューバー履歴・話者分離 |
| iOS | v1.1.0 | WidgetKit・Siri Shortcuts・Dynamic Island・無音検出改善 |

---

## 1. iPhoneとMacのローカルP2P連携（MultipeerConnectivity）

一番力を入れた機能がこれだ。**iPhoneで話した内容をBluetoothまたはWi-Fi経由でMacに直接送信する**。クラウドを経由しないし、ケーブルも要らない。

仕組みはAppleの`MultipeerConnectivity`フレームワーク。iPhone側が`MCNearbyServiceBrowser`で近くのMacを探し、Mac側が`MCNearbyServiceAdvertiser`で自分を広告する。接続確立後、iOSで文字起こしした結果をJSONとして送信する。

```swift
// Mac側: Advertiserを起動
let advertiser = MCNearbyServiceAdvertiser(
    peer: MCPeerID(displayName: Host.current().localizedName ?? "Mac"),
    discoveryInfo: nil,
    serviceType: "koe-bridge"
)
advertiser.startAdvertisingPeer()

// iPhone側: Browserで探す
let browser = MCNearbyServiceBrowser(
    peer: MCPeerID(displayName: UIDevice.current.name),
    serviceType: "koe-bridge"
)
browser.startBrowsingForPeers()
```

iOS側でハマったのが、**`NSLocalNetworkUsageDescription`と`NSBonjourServices`をInfo.plistに追加しないとローカルネットワークの許可ダイアログが出ない**という点だ。この2つが抜けていると、Bonjour探索がサイレントに失敗する。

```xml
<key>NSLocalNetworkUsageDescription</key>
<string>近くのMacと接続してテキストを送信するために使用します</string>
<key>NSBonjourServices</key>
<array>
    <string>_koe-bridge._tcp</string>
    <string>_koe-bridge._udp</string>
</array>
```

受け取ったテキストはMacのアクティブなテキストフィールドへそのままペーストされる。MacBookの内蔵マイクより口元に近いiPhoneの方が音質が良いことも多く、「iPhoneをマイクとして使ってMacで作業する」というユースケースが快適になった。

---

## 2. WidgetKit ウィジェット

ホーム画面から1タップで録音を開始できるウィジェットを追加した。small・medium・accessoryCircular・accessoryRectangularの4サイズに対応している。

```swift
struct KoeLaunchWidget: Widget {
    var body: some WidgetConfiguration {
        StaticConfiguration(kind: "KoeLaunchWidget", provider: Provider()) { _ in
            KoeWidgetView()
                .widgetURL(URL(string: "koe://transcribe"))
        }
        .configurationDisplayName("Koe 音声入力")
        .supportedFamilies([
            .systemSmall, .systemMedium,
            .accessoryCircular, .accessoryRectangular
        ])
    }
}
```

ウィジェットはアプリと別プロセスで動くため、`KoeRecordingAttributes`（ActivityKit用）をウィジェットターゲットにも含める必要がある。XcodeGenで管理しているので`project.yml`に`KoeWidget`ターゲットを追加した。

ロック画面ウィジェット（accessoryCircular）はApple Watchの文字盤感覚で使えて、地味に便利だ。

---

## 3. Siri Shortcuts（App Intents）

「Koeで音声入力」「声でメモ」と言えばアプリが起動して録音が始まる。`AppIntents`フレームワークの`AppShortcutsProvider`で実装した。

```swift
struct StartVoiceInputIntent: AppIntent {
    static let title: LocalizedStringResource = "音声入力を開始"
    static let openAppWhenRun = true

    @MainActor
    func perform() async throws -> some IntentResult {
        if let url = URL(string: "koe://transcribe") {
            await UIApplication.shared.open(url)
        }
        return .result()
    }
}

struct KoeShortcuts: AppShortcutsProvider {
    static var appShortcuts: [AppShortcut] {
        AppShortcut(
            intent: StartVoiceInputIntent(),
            phrases: [
                "Start voice input with \(.applicationName)",
                "\(.applicationName)で音声入力",
                "声でメモ"
            ],
            shortTitle: "音声入力を開始",
            systemImageName: "mic.fill"
        )
    }
}
```

`phrases`に`\(.applicationName)`を含めないとSiriに登録されないケースがあるので注意。またApp Shortcutsはアプリを最初に起動するだけで自動登録される（明示的な登録APIは不要）。

---

## 4. Dynamic Island / Live Activities

録音中はDynamic Islandにマイクアイコンと経過時間が表示される。`ActivityKit`で実装した。

```swift
struct KoeRecordingAttributes: ActivityAttributes {
    struct ContentState: Codable, Hashable {
        var isRecording: Bool
        var statusText: String
        var audioLevel: Double
    }
    var startTime: Date
}
```

ウィジェット側でDynamic Islandのコンパクト表示を定義する：

```swift
dynamicIsland: { context in
    DynamicIsland {
        // 展開時
        DynamicIslandExpandedRegion(.bottom) {
            Text(context.state.statusText)
        }
    } compactLeading: {
        Image(systemName: "mic.fill").foregroundColor(.red)
    } compactTrailing: {
        // 経過時間タイマー
        Text(context.attributes.startTime, style: .timer)
            .monospacedDigit()
            .font(.caption2)
    } minimal: {
        Image(systemName: "mic.fill")
    }
}
```

Info.plistに2つのキーが必要なのを忘れがち：

```xml
<key>NSSupportsLiveActivities</key>
<true/>
<key>NSSupportsLiveActivitiesFrequentUpdates</key>
<true/>
```

録音中にほかのアプリを開いても状況が一目で分かる。実装コスト対UX向上でもっとも満足度が高かった機能だ。

---

## 5. メニューバー履歴（Mac）

過去5件の音声入力をメニューバーから参照・コピーできる。`HistoryStore`でUserDefaultsに保存し、`rebuildMenu()`で履歴サブメニューを動的に生成している。

```swift
// AppDelegate.swift
let historyMenu = NSMenu()
for entry in HistoryStore.shared.entries.prefix(5) {
    let item = NSMenuItem(
        title: String(entry.text.prefix(40)),
        action: #selector(copyHistoryItem(_:)),
        keyEquivalent: ""
    )
    item.representedObject = entry.text
    historyMenu.addItem(item)
}
```

---

## 6. 話者分離（Mac版 議事録モード）

会議の録音で「誰が何を言ったか」を自動整理する。アプローチはシンプルで、**5秒以上の無音 → 話者切り替え**というルールベースだ。

```swift
// MeetingMode.swift
let speakerLabels = ["話者A", "話者B", "話者C", "話者D", "話者E"]

func append(text: String, gap: TimeInterval) {
    if gap >= speakerChangeThreshold { // 5.0秒
        currentSpeakerIndex = (currentSpeakerIndex + 1) % speakerLabels.count
    }
    let label = speakerLabels[currentSpeakerIndex]
    entries.append("[\(label)] \(text)")
}
```

声紋ベースの話者識別と比べると精度は劣るが、**ローカル処理・即時・依存ゼロ**という点では十分実用的。「発言後にちょっと間を置く」という自然な会話の流れを活かすと、かなり綺麗に分離できる。

---

## 7. 無音検出の改善（iOS）

以前はデフォルト3秒で自動停止していたが、**デフォルトを手動停止（999秒）に変更**した。「考えながら話す」「文章を組み立てる」という使い方で意図せず止まってしまうという声が多かったからだ。

設定画面で以下を調整できるようにした：
- **無音検出時間**: 1.5秒 / 3秒 / 6秒 / なし
- **感度**: 高（0.02）/ 中（0.005）/ 低（0.001）
- **最低録音時間**: 開始後3秒は自動停止しない

UserDefaultsに保存して`RecordingManager`で読み込む設計にしたので、設定変更が次の録音からすぐ反映される。

---

## サイトリニューアル

ランディングページも全面リニューアルした。 → https://koe.elio.love/

- 真っ黒背景 + 赤（#ff3040）アクセント
- 日本語/英語/中国語/韓国語を循環するタイプライターアニメーション
- パルスするマイクアイコン（CSSキーフレーム）
- 20本バーの波形アニメーション
- グラスモーフィズムカード
- Dynamic Islandのモックアップ表示
- IntersectionObserverによるスクロール連動アニメーション

フレームワーク不要の純粋なHTML/CSS/JSで実装して、Fly.io（nrt）にデプロイしている。

---

## おわりに

今回のアップデートで「iPhone → Mac」の音声ブリッジが実現し、手元のiPhoneがMacの高品質マイクとして機能するようになった。Dynamic IslandとWidgetKitの対応で、アプリを起動しなくても音声入力を始められる体験も整った。

次に取り組みたいのは：

- **リアルタイム翻訳**: 文字起こしと同時に別言語へ翻訳してMac画面に字幕表示
- **声紋ベース話者分離**: ルールベース → Core MLモデルへの移行

Koeは現在TestFlightで配布中（https://koe.elio.love/ からリンクあり）。機能リクエストや「こういう使い方をしてる」という声は [@yukihamada](https://twitter.com/yukihamada) まで。ぼく自身が毎日使うアプリなので、かゆいところに手が届くアップデートを続けていく。
