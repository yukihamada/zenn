---
title: "学術論文ベースのAIファクトチェッカーをRustで作ってOSSにした"
emoji: "🔍"
type: "tech"
topics: ["rust", "ai", "factcheck", "axum", "oss"]
published: true
---

## TL;DR

テキストを貼るだけで「どの部分が正確で、どの部分が嘘か」を**根拠付き**で判定するAIファクトチェッカー **[FactLens](https://factlens.co)** を作ってOSSにしました。

- **GitHub**: https://github.com/yukihamada/factlens
- **ライブ**: https://factlens.co
- **技術**: Rust + Axum + SQLite, 約1500行
- **ライセンス**: MIT

## ChatGPTに「これ正しい？」と聞いてはいけない理由

試しにやってみてください。ChatGPTに「人間の脳は10%しか使われていない、これは正しいですか？」と聞くと、モデルやタイミングによっては「はい、よく知られた事実です」と返ってきます。

**これは明確に間違い**です。神経科学の教科書（Kandel et al., *Principles of Neural Science*）は脳全体が常に活動していることを示しています。

LLMでファクトチェックする根本的な問題は3つ:

| 問題 | 具体例 |
|------|-------|
| **ハルシネーション** | 存在しない論文を引用して「研究で証明されています」と回答 |
| **知識カットオフ** | 2024年以降のニュースは「わかりません」としか言えない |
| **根拠なし** | 「科学的に否定されています」←どの科学？何の研究？ |

Huang et al. (2023) "A Survey on Hallucination in Large Language Models" でこの問題は体系的に報告されています。

**AIでファクトチェックしたいのに、AI自体が嘘をつく。** この矛盾をどう解決するか？

## 解法: Web検索 × 原子的分解 × 出典強制

FactLensは「LLMに直接聞く」のではなく、**4段階のパイプライン**で検証します。各段階に査読付き論文の裏付けがあります。

```
ユーザー入力
    │
    ▼
┌─────────────────────────────┐
│ 1. リアルタイムWeb検索        │ ← RAG (Lewis et al., NeurIPS 2020)
│    Jina AI Searchでエビデンス取得│
└──────────┬──────────────────┘
           ▼
┌─────────────────────────────┐
│ 2. 原子的事実に分解           │ ← FActScore (Min et al., EMNLP 2023)
│    "1889年建造で高さ500m"     │
│    → claim_1 + claim_2      │
└──────────┬──────────────────┘
           ▼
┌─────────────────────────────┐
│ 3. 出典付きで各claimを判定    │ ← Chain-of-Thought (Wei et al., 2022)
│    supported / refuted /     │    + FEVER 3ラベル体系 (Thorne et al., 2018)
│    unverifiable              │
└──────────┬──────────────────┘
           ▼
┌─────────────────────────────┐
│ 4. 加重スコア (0〜100)        │
│    supported=100 / unverifiable=50 / refuted=0│
└─────────────────────────────┘
```

### ステップ1: Web検索でグラウンディング

LLMの学習データに頼らず、最新のWebソースを判定根拠として注入します。Lewis et al. (NeurIPS 2020) の **RAG (Retrieval-Augmented Generation)** を検証タスクに適用。

```rust
async fn web_search(query: &str) -> String {
    let client = reqwest::Client::builder()
        .user_agent("FactLens/1.0 (fact-checking bot)")
        .timeout(std::time::Duration::from_secs(10))
        .build()
        .unwrap_or_default();

    let encoded = urlencoding::encode(query);
    let url = format!("https://s.jina.ai/{encoded}");

    match client
        .get(&url)
        .header("X-Return-Format", "text")
        .send()
        .await
    {
        Ok(resp) if resp.status().is_success() => {
            let text = resp.text().await.unwrap_or_default();
            text.chars().take(3000).collect() // 先頭3000文字
        }
        _ => String::new(),
    }
}
```

**なぜJina AI Search？**
- APIキー不要、無料
- Google/Bing/DuckDuckGoはクラウドIPからCAPTCHAでブロックされる（Lambda/Fly.ioから叩けない）
- Jina はボット前提の設計で安定

### ステップ2: 原子的事実への分解 (FActScore)

Meta AIのMin et al. (EMNLP 2023, **Best Paper**) が提唱した手法。文章を「個別に検証可能な最小単位のクレーム」に分解します。

> 入力: 「エッフェル塔は1889年に建設され、高さは500メートルである」
>
> → claim_1: 「エッフェル塔は1889年に建設された」 → ✅ supported
> → claim_2: 「エッフェル塔の高さは500メートルである」 → ❌ refuted (実際は330m)

文全体に「半分正しい」とラベルを付けるより、**どの部分が正確でどの部分が間違っているか**が明確になります。

### ステップ3: 出典強制プロンプト

ここが精度のカギ。「科学的に否定されている」のような曖昧な根拠を**プロンプトレベルで禁止**しています。

```rust
fn build_prompt(text: &str, search_context: &str) -> String {
    format!(r#"You are FactLens, a rigorous expert fact-checking AI.

WEB SEARCH RESULTS (use as primary evidence):
{search_context}

TEXT TO ANALYZE:
{text}

REASONING QUALITY REQUIREMENTS — CRITICAL:
• Always cite a SPECIFIC source, institution, study
• Good: "WHO 2023年報告書によると…" "Nature誌2019年の研究では…"
• Bad: "科学的に否定されている" / "専門家の見解では"
• Include: journal names, organization names, publication years

Respond ONLY with valid JSON:
{{"summary": "...", "atomic_facts": [
  {{"claim": "...", "verdict": "supported", "reasoning": "具体的出典..."}}
]}}"#)
}
```

Wei et al. (NeurIPS 2022) の **Chain-of-Thought** の知見を応用し、中間推論を明示させることで判定品質を上げています。

### ステップ4: 加重スコアリング

```rust
fn compute_result(output: FactCheckOutput, provider: &str) -> CheckResult {
    let total = output.atomic_facts.len();
    let score = if total == 0 {
        50u8
    } else {
        let points: usize = output.atomic_facts.iter().map(|f| {
            match f.verdict.as_str() {
                "supported" => 2,      // 100点相当
                "unverifiable" => 1,   // 50点相当
                _ => 0,                // refuted = 0点
            }
        }).sum();
        ((points as f32 / (total * 2) as f32) * 100.0) as u8
    };
    // ...
}
```

**なぜ `unverifiable` を50点にするか？**

`unverifiable` を無視すると、「1つ supported + 5つ unverifiable」が100点になってしまう。政治的主張のように検証困難なクレームが多い文章が、根拠なく高スコアを得ることを防ぐ設計です。

## 実際の判定例

入力:
> 「適度なアルコールは健康に良い。人間の脳は10%しか使われていない。金魚の記憶は3秒しかもたない。」

結果: **スコア 0点 (FALSE)**

| クレーム | 判定 | 根拠 |
|---------|------|------|
| 適度なアルコールは健康に良い | ❌ refuted | WHO 2023年報告書「アルコールに安全な量はない」、Lancet 2018年の大規模研究 |
| 人間の脳は10%しか使われていない | ❌ refuted | 神経科学教科書 Kandel et al., fMRI研究で脳全体の活動を確認 |
| 金魚の記憶は3秒しかもたない | ❌ refuted | Plymouth大学2003年の研究、金魚は数ヶ月の記憶保持を実証 |

3つとも有名な科学的俗説で、すべて具体的な出典付きで否定されます。

## 技術スタック

| コンポーネント | 技術 | 選定理由 |
|------------|------|---------|
| Backend | **Rust + Axum 0.7** | 512MB VMで余裕で動作。コールドスタート数百ms |
| DB | **SQLite (WAL mode)** | 設定ゼロ、ファイル1つで完結 |
| Web検索 | **Jina AI Search** | 無料・APIキー不要・クラウドIPでもブロックされない |
| デフォルトLLM | **Kimi-K2 via Groq** | MoE 1Tパラメータ、無料枠あり、高速推論 |
| 認証 | **Argon2** | メモリハード関数、業界標準 |
| デプロイ | **Fly.io 東京** | SQLiteボリューム対応、auto_stop で省コスト |

**全体で約1500行**。フレームワークに頼らず、HTMLテンプレートもRust内でSSRしています。

なぜRust？ → Fly.ioの512MB VMで余裕で動くメモリ効率と、`auto_stop_machines = "stop"` でアイドル時にマシンを停止→リクエスト時に数百msで起動できるコールドスタート性能が決め手。Pythonだと起動に数秒かかる。

## 機能一覧

- 🌐 **Webアプリ**: テキスト or URL貼り付けで即チェック
- ⌨️ **CLI**: `cargo run --bin factlens -- "万里の長城は宇宙から見える"`
- 🔌 **公開API**: `POST /api/v1/check` でプログラムから利用
- 🧩 **埋め込みウィジェット**: `<iframe src="factlens.co/embed/{id}">` でブログに貼れる
- 🔀 **マルチモデル比較**: Claude / Kimi-K2 / Llama / Qwen3 を並列実行
- 🔗 **シェア**: パーマリンク + OGP画像 + X/LINE シェアボタン
- 🧩 **Chrome拡張**: 選択テキストを右クリックでチェック
- 💬 **LINE Bot**: LINEでテキスト送信→即ファクトチェック
- 🏫 **教育ページ**: 50分授業モデル付き ([/education](https://factlens.co/education))
- 🛡️ **PII保護**: メール・電話番号・カード番号を自動検出・マスク

## セルフホスト

```bash
git clone https://github.com/yukihamada/factlens.git
cd factlens
cp .env.example .env
# .envにGROQ_API_KEY（無料: https://console.groq.com/）を設定
cargo run --bin factlens-web
# → http://localhost:8080
```

Dockerでも:
```bash
docker build -t factlens .
docker run -p 8080:8080 -e GROQ_API_KEY=gsk_xxx -v factlens_data:/data factlens
```

## APIの使い方

```bash
# ファクトチェック実行
curl -X POST https://factlens.co/api/v1/check \
  -H "Content-Type: application/json" \
  -d '{"text": "人間の脳は10%しか使われていない"}'

# レスポンス
{
  "id": "abc123",
  "score": 0,
  "label": "FALSE",
  "summary": "神経科学の確立された知見に反する主張です",
  "atomic_facts": [...],
  "url": "https://factlens.co/r/abc123",
  "embed_url": "https://factlens.co/embed/abc123"
}
```

## 参考論文

FactLensの手法は以下の査読付き論文に基づいています:

1. **FActScore** — Min, S. et al. "Fine-grained Atomic Evaluation of Factual Precision in Long Form Text Generation." *EMNLP 2023* (Best Paper)
2. **RAG** — Lewis, P. et al. "Retrieval-Augmented Generation for Knowledge-Intensive NLP Tasks." *NeurIPS 2020*
3. **FEVER** — Thorne, J. et al. "A Large-scale Dataset for Fact Extraction and VERification." *NAACL 2018*
4. **Chain-of-Thought** — Wei, J. et al. "Chain-of-Thought Prompting Elicits Reasoning in Large Language Models." *NeurIPS 2022*
5. **Hallucination Survey** — Huang, L. et al. "A Survey on Hallucination in Large Language Models." *2023*
6. **LLM Fact-Checking** — Quelle, D., Bovet, A. "The Perils and Promises of Fact-Checking with LLMs." *Frontiers in AI, 2024*

## 限界

正直に書きます:

- **検索結果の質に依存**: Web上の誤情報がそのまま根拠になる可能性
- **LLMのバイアス**: 学習データの偏りが判定に影響
- **日本語の情報量**: 英語に比べて検索結果の質が落ちる場合あり
- **専門分野**: 医学・法律は一般Web検索では証拠不足

だからこそ、「最終判定」ではなく「初期スクリーニング」として位置づけています。人間の本格的なファクトチェックは数時間かかりますが、FactLensはその最初の一歩を数秒で提供します。

## まとめ

「AIにファクトチェックさせたいけど、AI自体が嘘をつく」——この矛盾を、**Web検索によるグラウンディング**と**原子的事実分解**で解決するのがFactLensのアプローチです。

コード全体をMITライセンスで公開しています。プロンプトもスコアリングロジックも全部見れます。ファクトチェッカーこそ透明であるべきだと思っているので。

フォーク、PR、Issue、なんでも歓迎です。

**https://factlens.co**
**https://github.com/yukihamada/factlens**
