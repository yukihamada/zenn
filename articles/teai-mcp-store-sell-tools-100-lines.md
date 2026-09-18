---
title: "自作ツールをAIエージェントに従量課金で売る — Cloudflare Workers 100行でMCPストアに出品するまで"
emoji: "🧰"
type: "tech"
topics: ["mcp", "cloudflareworkers", "ai", "claudecode", "api"]
published: true
---

自作のツールを、AIエージェントに従量課金で売る。そのための最小手順を、実測ベースで書きます。Cloudflare Workers に JSON-RPC の POST ハンドラを1本置いて、teai.io の MCP ツールストアに登録するまで。コードは約100行です。

先に正直な現状を書いておきます。**このストアに並んでいるツールは、いま全部筆者(運営)が自分で出したものです。外部の開発者はまだ0人。** だからこそ、最初の数人には筆者が実装から出品・課金確認まで直接伴走します。この記事はその招待状でもあります。

## 何ができるか

```
あなたのMCPサーバ(どこでもOK・公開HTTPS必須)
   ↑ tools/list, tools/call (JSON-RPC 2.0 over HTTP POST)
teai.io ゲートウェイ  https://api.teai.io/mcp/{あなたのslug}
   ↑ teaiキーで認証・呼び出しごとにクレジット課金
利用者(AIエージェント / Claude Code / Sente ほか)
```

- 課金は teai 側が行い、ツールが呼ばれるたびに**消費クレジットの80%を基準に開発者の台帳へ記帳**されます(1回ごとに切り上げ)
- 価格は登録時に自分で決めます(1〜60クレジット/回)
- 上流エラー時・残高不足時は課金されません(誤課金なしは実測で確認)
- 分配・出金の規約は**法務確認前のドラフト**です。ここは盛らずに書きます。現行の条件は登録ページで確認してください

## 1. 雛形をコピーする(約100行)

公開している雛形: https://github.com/yukihamada/teai-docs/tree/master/mcp-starter

必要なのは JSON-RPC 2.0 の POST ハンドラ1本。実装必須のメソッドは4つだけです。

| メソッド | 返すもの |
|---|---|
| `initialize` | protocolVersion, capabilities, serverInfo |
| `notifications/*` | 202 で受け流す |
| `tools/list` | ツール定義の配列(name / description / inputSchema) |
| `tools/call` | `{content: [{type:"text"|"image"|"resource", ...}]}` |

雛形の中で書き換えるのは `TOOLS`(ツール定義)と `handleToolCall()`(処理本体)の2箇所だけです。以下は雛形そのままの「文字数カウント」ツール。

```js
const TOOLS = [
  {
    name: "count_chars",
    description:
      "テキストを渡すと、文字数・単語数・行数をJSONで返します。\n" +
      "入力: text(必須・100,000文字まで)。\n" +
      "出力: {chars, words, lines}。\n" +
      "実測レイテンシ: 0.1秒未満。\n" +
      "できないこと: 形態素解析はしません(単語数は空白区切りの概算)。",
    inputSchema: {
      type: "object",
      properties: { text: { type: "string", maxLength: 100000 } },
      required: ["text"],
    },
  },
];

function handleToolCall(name, args) {
  if (name === "count_chars") {
    const text = args?.text;
    if (typeof text !== "string" || text.length === 0) throw rpcError(-32602, "text は必須です");
    if (text.length > 100000) throw rpcError(-32602, "text は100,000文字までです");
    return { content: [{ type: "text", text: JSON.stringify({
      chars: [...text].length,
      words: text.trim().split(/\s+/).filter(Boolean).length,
      lines: text.split("\n").length,
    }) }] };
  }
  throw rpcError(-32601, `unknown tool: ${name}`);
}
```

残りの JSON-RPC ボイラープレート(initialize / tools/list / tools/call の振り分け、パースエラー処理)は雛形に入っています。触らなくて動きます。

## 2. 説明文の型 — ここが一番効く

エージェントは**説明文と価格だけを読んで道具を選びます**。人間向けのブランディングは効きません。効くのはこの型です。

```
1行目: 「◯◯を渡すと、◯◯を返します」— 具体的に
入力: 各パラメータの意味・単位・上限・既定値(例つき)
出力: 形式とサイズ感
実測レイテンシ: 「実測でおよそN秒」(必ず自分で3回測る)
できないこと: 限界を正直に(OCRしない・実データ照合しない等)
```

盛らないこと。「5分で」「完璧に」のような未検証の形容は書かない。説明文の通りに動かないツールを、エージェントは二度と呼びません。

## 3. 価格の決め方(筆者の実例)

| クラス | 目安 | 実例 |
|---|---|---|
| 軽量ユーティリティ(ms〜1秒) | 3〜5cr | QRコード生成=3cr, PDFテキスト抽出=5cr |
| 生成系(1〜5秒) | 10〜15cr | OGP画像生成=10cr, 請求書PDF=15cr |
| 重いAI呼び出し(数十秒) | 30〜60cr | 商標リスク診断=50cr |

分配は切り上げ式なので、低額ツールは開発者の取り分がほぼ100%になります(3cr→3cr)。3cr を下回る価格設定は意味が薄いです。

## 4. デプロイして登録する

```bash
npm i -g wrangler && wrangler login      # npx wrangler はハングすることがあるのでグローバル推奨
git clone --depth 1 https://github.com/yukihamada/teai-docs
cp -r teai-docs/mcp-starter my-tool && cd my-tool
# wrangler.toml の name と src/index.js の TOOLS / handleToolCall を書き換える
wrangler deploy
# 20〜30秒待ってから、5回連続で200を確認(エッジ伝播ラグ)
curl -s https://my-tool.<you>.workers.dev -d '{"jsonrpc":"2.0","id":1,"method":"tools/list"}'
```

登録(無料のteaiキーが必要。登録時に上流へ tools/list の実疎通が走るので、先にデプロイしておく):

```bash
curl -X POST https://api.teai.io/api/v1/mcp/services \
  -H "Authorization: Bearer $TEAI_API_KEY" -H 'Content-Type: application/json' \
  -d '{"slug":"mytool","upstream_url":"https://my-tool.<you>.workers.dev","default_credits":5}'
# → {"status":"pending"} 運営が審査して live に(最初の数人は当日中に返します)

curl https://api.teai.io/api/v1/mcp/services/mine  -H "Authorization: Bearer $TEAI_API_KEY"   # 状態
curl https://api.teai.io/api/v1/mcp/earnings       -H "Authorization: Bearer $TEAI_API_KEY"   # 台帳
```

フォームから登録したい場合はこちら: https://teai.io/mcp-developers

## 5. 実例: 公開ページだけで動くツール(検索0.45秒)

外部データもDBも無しで、あるサイトの sitemap と OGP だけを読んで「街名 → その街のイラストトート最大5件」を返すツールを、雛形ベースで約120行・30分で書きました。Node で直接ハンドラを叩いたローカル実測は、検索 452ms・最新1件 192ms。手持ちの公開ページやAPIがあれば、それをそのまま道具にするのが最短です。

## 6. 実測で踏んだ罠(全部一次情報)

1. **上流レスポンス5MB上限** — フォント全埋め込みPDF(4.2MB)を base64 で二重出力して超過。バイナリは1回だけ出力・フォントは subset 埋め込み(4.2MB→35KB)
2. **slug は3文字以上** — "qr" は拒否された→ "qrcode"
3. **デプロイ直後20〜30秒はエッジ伝播ラグ**で 1042/1104 エラー。5回連続200を確認してから登録
4. **課金検証は残高でなく台帳で** — 共有アカウントだと他プロセスの消費が混ざる。`/api/v1/mcp/earnings` が正本
5. **JSON-RPC エラー(HTTP 200内)を返せば利用者に課金されない** — 入力バリデーションはきちんと JSON-RPC エラーで返す。それが信頼になる
6. 登録APIはレート制限5回/分
7. `wrangler dev` のローカル workerd は compatibility_date の未来日付を拒否することがある(本番は問題ない)
8. curl の素朴な計測は TLS ハンドシェイク遅延を含む。レイテンシはハンドラ内で測るか、複数回の中央値を取る

## 出品前チェックリスト

- [ ] tools/list が説明文の型に沿っている(レイテンシ実測済み・できないこと明記)
- [ ] 変な入力(空・超過・型違い)が JSON-RPC エラーで返る
- [ ] レスポンスが 5MB 未満
- [ ] 本番URLで5回連続200
- [ ] live 後、ゲートウェイ経由で1回実呼び出しして台帳に記帳されることを確認

## おわりに — 最初の1人を探しています

もう一度、正直に。外部の開発者はまだ0人です。最初の数人には、何を作るかを一緒に1本に絞るところから、雛形の調整、登録、審査(当日)、初回の課金が台帳に載るところまで、筆者が直接伴走します。すでに公開している MCP サーバーがあるなら、HTTPS の URL を登録するだけで出品できます。

- 登録ページ: https://teai.io/mcp-developers
- 詳しい開発者ガイド: https://teai.io/blog/mcp-dev-guide
- 雛形: https://github.com/yukihamada/teai-docs/tree/master/mcp-starter

「これ出せる?」だけでも、登録ページの連絡先か筆者のXまで。
