---
title: "caffeinateは蓋閉じを防げない — AIエージェントを常駐VMに移した実測"
emoji: "🖥"
type: "tech"
topics: ["mac", "flyio", "tmux", "ai", "個人開発"]
published: true
---

:::message
筆者は [teai.io](https://teai.io) を運営する株式会社イネブラの代表です。当事者の実装記録として読んでください。数値はすべて 2026-09-14 時点の本番環境での実測値です。
:::

## 3行で

- 「Mac のふたを閉じてもエージェントを止めたくない」は、**macOS の設定では解決できない**
- `caffeinate` が 321時間稼働していても、蓋を閉じた瞬間に止まる（ハードウェアレベルの強制スリープ）
- なので実行場所を Mac から常駐 VM に移した。実ジョブ 4件すべて成功、所要 20〜25秒、約 $5/月

## 最初に結論：lid close は設定で止められない

いきなり詰まりどころから書きます。ここを飛ばすと数時間溶かします。

最初に調べたとき、手元の Mac ではすでに `caffeinate -dims` が **321時間**稼働していて、`PreventSystemSleep` も有効でした。つまりアイドルスリープは完全に抑止できていた。それでも蓋を閉じれば終わりです。

| 手段 | アイドル | 蓋閉じ | 備考 |
|---|---|---|---|
| `caffeinate -dims` | 防げる | **不可** | 321時間稼働中でも無効だった |
| `pmset disablesleep` | 防げる | **不可** | Apple Silicon では lid close に効かないという報告が多い |
| クラムシェル（AC + 外部ディスプレイ） | 防げる | 防げる | 確実だが Mac を動かせない |
| **サーバーで動かす** | 無関係 | 無関係 | **PC の状態に依存しない** |

macOS の lid close は `pmset` や `caffeinate` の外側、ハードウェアレベルで強制されます。クラムシェルは確実ですが、ノートを閉じて移動したいという元の動機に合いません。

というわけで、**PC の電源状態に依存させない唯一の方法は、実行場所を PC の外に出すこと**でした。

## 構成：VM が主役、Mac は依頼者

役割をはっきり分けました。実行するのは常駐 VM、依頼するのは Mac（またはスマホ）です。

- **Fly.io `sente-cloud`**（東京リージョン / shared-cpu-2x 2GB / volume 10GB）— エージェントが tmux セッション `sente` で常駐。ジョブを拾う worker が `flyworker` で待機
- **Cloudflare Worker** — `POST /jobs` に実行先 `fly` を追加。VM が `/jobs/fly/next` をポーリングして拾う
- **結果** — volume に残るので、VM を作り直しても作業は消えない

![fly status の実画面。machine が started、volume sente_data が接続されている](https://teai.io/blog-assets/fly-resident-01-status.png)
*`fly status -a sente-cloud` — machine `891e035c676598` が `started`。volume `sente_data`（10GB）が `/home/yuki/workspace` に接続済み。*

「Mac を閉じても止まらない」理由は単純です。エージェントは tmux の中で動き、sshd がコンテナの主プロセス。さらに `auto_stop_machines = false` と `min_machines_running = 1` で、アイドルによる自動停止も無効化しています。

![Fly VM 上で動いているエージェントの TUI 画面](https://teai.io/blog-assets/fly-resident-02-tui.png)
*VM の中で動いているエージェントの TUI。MCP 2個を認識し、provider 選択画面になっていない（＝設定が正しく読めている）。*

## 実際に使ってみる

「動きました」ではなく、ファイルを作らせました。

ジョブは `target: "fly"` を付けて投げるだけです。Mac は依頼を出したら、あとは何もしません。

```bash
$ curl -X POST https://sente.teai.io/jobs \
    -H 'content-type: application/json' \
    -d '{"target":"fly",
         "prompt":"/home/yuki/workspace に hello-fly.md を作って
                  「Fly VM で動きました」と1行書いて。
                  その後 cat で内容を確認して報告して。"}'
```

![ジョブを投げてから結果を取得するまでの実画面](https://teai.io/blog-assets/fly-resident-04-job.png)
*投げてから約24秒後、`status: done`・`rc: 0`。VM が自分でファイルを書き、`cat` で確認して報告している。*

次はもう少し実務寄りの依頼です。システム情報を調べて Markdown の表にまとめさせました。

```json
"prompt": "/home/yuki/workspace に report.md を作って。
   今日の日付・ホスト名・CPUコア数・メモリ量・ディスク使用量を
   uname/hostname/nproc/free/df で調べて、Markdown の表にまとめて。
   最後に cat で中身を見せて。"
```

![VM が自分で作った report.md の内容](https://teai.io/blog-assets/fly-resident-05-report.png)
*VM が `uname` / `hostname` / `nproc` / `free` / `df` を実行し、自分で Markdown の表にまとめた結果。*

![workspace に成果物が残っている実画面](https://teai.io/blog-assets/fly-resident-06-ls.png)
*`hello-fly.md` と `report.md` が volume 上に残る。VM を作り直しても消えない。*

| 項目 | 実測 |
|---|---|
| 成功したジョブ | **4/4**（すべて `rc=0`） |
| ジョブ所要時間 | **20〜25秒**（ファイル作成＋確認まで） |
| 追加の推論コスト | **$0**（既存の残高をそのまま使用） |
| VM の維持費 | 約 **$5/月**（shared-cpu-2x 2GB + 10GB volume） |

## 踏んだ罠：7つすべて本番で実測

実装自体は半日でしたが、**本番で動かして初めて分かる問題が7つ**ありました。どれもログだけでは「成功」に見えるのが厄介です。

1. **worker トークンが 403** — `WORKER_PATHS` に `/jobs/fly/*` を入れ忘れ
2. **output 報告が 403** — `canReportJob` が `target !== 'mac'` で拒否。heartbeat だけ通るので「動いているように見える」
3. **全ジョブが「こちらはデモです」** — worker が `TEAI_API_KEY` を export できていない。**`rc=0` なので成功に見える最悪のパターン**
4. **`ProviderNoProvidersError`** — 生のバイナリは `~/.config/sente/config.json` を読む。ランチャー経由だと動くので気づかない
5. **agent "sente" not found** — VM に実在するのは `build`（primary）。手元と同じ名前が VM にあるとは限らない
6. **`sente run -p` は password** — prompt ではない。prompt は positional で渡す
7. **volume が root 所有** — 初回作成時に root 所有になり、workspace に書き込めない。エージェントが「権限がない」と判断して作業を止めた

:::message alert
**教訓：終了コード 0 を信じない。**
3番目の罠では、worker のログに `rc=0 status=done` と出ていました。中身は「こちらはデモです」です。API キーが空でもエージェントは正常終了するため、**出力の中身を見るまで分かりません**。今はキーが空なら worker が起動を止めるようにしています。
:::

これ、AI エージェントの自動化全般に言える話だと思っています。exit code は「プロセスが落ちなかった」ことしか保証しない。エージェントが**何もしなくても** 0 で終わる世界では、成功判定を別に持たないと事故ります。

## Mac からどう使うか

**1. 直接入って操作する** — 手元の代わりに VM のエージェントを触りたい場合。

```bash
fly ssh console -a sente-cloud   # 入る（root）
su - yuki                        # 実行ユーザに切替
tmux attach -t sente             # エージェントの画面に入る
```

`Ctrl+B` → `D` で離脱しても、**エージェントは動き続けます**。これが「Mac を閉じても止まらない」の実体です。

**2. ジョブとして投げる** — スマホやブラウザから依頼する場合。`target: "fly"` を付けるだけです。従来の `"mac"`（Mac が拾う）や `"cloud"`（サンドボックスで実行）と並んで選べます。

## これは「ローカル実行は要らない」という話ではない

誤解されたくないので書きます。Mac でしかできない仕事（実ブラウザの操作、手元のファイル、キーチェーン）は今までどおりローカル実行です。変えたのは**「PC の電源状態に依存させたくない仕事」の置き場所**だけです。

未検証の点も残しています。

- 長時間ジョブ（タイムアウト 3600 秒）の実走はまだ。今回は 20〜25 秒のジョブしか試していない
- worker トークンに有効期限があり、失効したら再発行が必要

## まとめ

「PC を起動し続ける」問題は、**PC の外に出せば消える**。スリープを抑止する設定を頑張るより、実行場所を移す方が早く、確実で、結果的に安い。常駐 VM は約 $5/月です。

同じことを考えている人の参考になれば幸いです。実測の内訳は [teai.io のブログ](https://teai.io/blog/sente-fly-resident)にもう少し詳しく書いています。
