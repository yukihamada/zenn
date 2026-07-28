---
title: "teai.ioに都度チャージ(pay-as-you-go)機能を実装した話"
emoji: "💳"
type: "tech"
topics: ["stripe", "rust", "api", "saas"]
published: false
---

## はじめに

[teai.io](https://teai.io) はこれまで、Free / Pro（¥4,350/月）/ Business（¥14,800/月）という月額サブスクリプション中心の料金体系でやってきた。Freeプランでもサインアップ時に1,000 credits・Nemotron 9Bは無制限無料なので、まず試すハードルは低い。

ただ実際に使ってもらう中で、「サブスクを組むほどではないが、あと少しだけcreditsが欲しい」というニーズが見えてきた。月末にPro枠を使い切ったが更新まで数日ある、個人の検証用途で月額契約は大げさ、といったケースだ。ここを埋めるために**都度チャージ（pay-as-you-go）**を実装した。

## なぜ月額サブスクのみだと機会損失か

月額プランは「継続利用が見込める人」には合理的だが、次のような層を取りこぼす。

- ちょっと試したいだけで、月額契約という意思決定コストを払いたくない人
- Free枠のcreditsを使い切ったが、次のプラン更新まで待てない人
- 法人の経費精算で「都度払い」の方が通しやすいケース

サブスクへの導線しかないと、これらの人は離脱するかFreeのまま留まる。少額から始められる決済導線を用意することで、この機会損失を減らせる。

## 実装

Stripe Checkoutの one-time payment モードでチャージ用セッションを発行する構成にした。サブスクと違い、支払いが完了した時点で `checkout.session.completed` webhookを受け、その場でcreditsを加算して終わりというシンプルなフローになる。

既存のStripe連携はasync-stripe crateではなく `reqwest` でREST APIを直接叩く自前実装になっているため、今回もそれに揃えた（概略）。

```rust
// crates/nanobot-core/src/service/stripe.rs（概略、実際はreqwestでform POST）
pub fn build_topup_checkout_params(amount_jpy: u32, user_id: &str) -> Vec<(String, String)> {
    let credits = topup_credits_for_amount(amount_jpy); // 1円 = 6 credits
    vec![
        ("mode".into(), "payment".into()), // subscriptionではなくone-time
        ("line_items[0][price_data][currency]".into(), "jpy".into()),
        ("line_items[0][price_data][unit_amount]".into(), amount_jpy.to_string()),
        ("metadata[type]".into(), "credit_topup".into()),
        ("metadata[teai_user_id]".into(), user_id.into()),
        ("metadata[credits]".into(), credits.to_string()),
    ]
}
// handle_topup_checkout 内で reqwest::Client でこのフォームを Stripe Checkout Sessions API に POST する
```

webhook側では `metadata.type == "credit_topup"` で判定し、サブスクの `checkout.session.completed`（テナント作成用）の経路とは分岐させている。ただし**これは経路を分けただけで、月次のクレジットリセット処理（`invoice.paid` 時に `credits_remaining` を月間許容量へ上書きする既存ロジック）との相互作用は別問題**で、まだ検証しきれていない。Pro/Business会員が都度チャージした直後に月次更新が走った場合の挙動は、要確認事項として残っている（現状このリセット処理自体が `dynamodb-backend` feature限定で本番の `libsql-backend` ビルドではコンパイルされておらず動いていないため、今のところ実害はない）。金額の選択肢は固定の5段階に絞り、`web/teai-pricing.html` からモーダル的にCheckoutへ飛ばす導線にしている。

## 金額とcredits換算

換算レートは月額プランと揃え、**1円 = 6 credits** とした。

| チャージ額 | 付与credits |
|---|---|
| ¥500 | 3,000 |
| ¥1,000 | 6,000 |
| ¥2,000 | 12,000 |
| ¥5,000 | 30,000 |
| ¥10,000 | 60,000 |

Proプラン（¥4,350で30,000 credits＝¥0.145/credit）と比べると、都度チャージ（1円=6credits＝¥0.167/credit）は約15%割高になる。継続利用が見込めるならProの方がお得、単発・少額ならその分の手軽さで都度チャージを選ぶ、という素直な棲み分けにした。

## 現状

実装自体は完了しており、PR #133（`stripe.rs` / `http.rs` / `teai-pricing.html`）としてレビュー待ちの状態だ。まだ本番にはマージされていないので、記事公開時点ではまだ使えない。マージ後、既存のサブスクフローと干渉しないことを確認してから本番に出す予定。

## おわりに

サブスクは「続ける前提の人」向け、都度チャージは「まず使ってみたい人」向け。両方の入り口を用意することで、Freeプランで止まっていた層にもう一段先を試してもらえるようにしたい。

マージ・本番反映され次第、改めて使い方を書く予定だ。
