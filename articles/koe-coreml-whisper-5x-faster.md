---
title: "Mac音声入力アプリ「Koe」をCoreML対応したら認識速度が5倍になった"
emoji: "🎙"
type: "tech"
topics: ["whisper", "coreml", "swift", "macos", "音声認識"]
published: true
---

**TL;DR**: whisper.cppベースのMac音声入力アプリ [Koe](https://koe.elio.love) で、CoreMLエンコーダを有効化したら音声認識が約5倍高速化した。Metal GPU onlyで5秒かかっていた処理が1秒に。やったことと、ハマったポイントを記録する。

---

## 自分が使うプロダクトは自分で作る

僕は日常的に使うツールをできる限り自分で作るようにしている。音声入力もそのひとつだ。

理由は2つ。

**安全性**。音声入力は自分の発言をすべてテキスト化する。仕事の会話、個人的な内容、パスワードの読み上げ——何がマイクに入るかわからない。クラウドに音声を送るサービスを信用するかどうかの問題ではなく、そもそも送らないのが一番安全だ。Koeは完全ローカルで動く。音声データは一切外に出ない。

**信頼性**。自分でコードを読めるプロダクトは、動作を信頼できる。何が起きているか分からないブラックボックスに依存するのは不安だし、問題が起きたとき自分で直せない。だからKoeはオープンソースにしている。誰でもコードを読めるし、フォークして自分用に改造できる。

この考え方はKoeだけではない。[Elio](https://elio.love)（オフラインAIチャット）、[OpenClaw](https://github.com/openclaw/openclaw)（自律AIエージェント）、どれも同じ思想で作っている。自分のデータは自分の手元に。使うツールのコードは自分で把握する。

---

## Before / After

| 条件 | Metal GPU only | CoreML + Metal |
|------|---------------|----------------|
| 0.5秒の音声 | ~3-5秒 | **1.0秒** |
| 6秒の音声 | ~5.3秒 | ~1.5秒 |
| モデル | Large V3 Turbo Q5 | 同じ |
| チップ | Apple M2 | 同じ |

体感が全く違う。話し終わった瞬間にテキストが出る。

## Koeとは

[Koe（声）](https://github.com/yukihamada/Koe-swift)は、Macで動く完全ローカルの音声入力アプリ。whisper.cppをSwiftから直接呼び出して、ショートカットキー（⌥⌘V）を押している間の音声をリアルタイムでテキスト変換する。

- whisper.cpp + Metal GPU で完全オフライン
- 20言語対応、言語別に最適モデルを自動選択
- Apple Speech APIでリアルタイムプレビュー → whisperで高精度確定
- 無料・オープンソース

```bash
# インストール
curl -fsSL https://koe.elio.love/install.sh | bash
```

## なぜCoreMLか

whisper.cppはMetal（GPU）で推論できるが、Apple SiliconにはもうひとつANE（Apple Neural Engine）がある。M2のANEは15.8 TOPS。CoreMLを経由すればANEが使える。

whisper.cppのCoreMLサポートは、**エンコーダ部分だけ**をCoreMLで実行する仕組み。Whisperの処理時間の大部分はエンコーダなので、ここをANEに載せるだけで劇的に速くなる。

## やったこと

### 1. whisper.cppをCoreML有効でビルド

```bash
cd /tmp
git clone https://github.com/ggml-org/whisper.cpp
cd whisper.cpp && git checkout v1.8.3

mkdir build && cd build
cmake .. \
  -DCMAKE_BUILD_TYPE=Release \
  -DWHISPER_COREML=ON \
  -DWHISPER_METAL=ON \
  -DGGML_METAL=ON
make -j8
```

これで `libwhisper.dylib` と `libwhisper.coreml.dylib` が生成される。

### 2. CoreMLモデルの変換

```bash
cd /tmp/whisper.cpp

# Python依存: pip install openai-whisper coremltools
python3 models/convert-whisper-to-coreml.py \
  --model large-v3-turbo \
  --encoder-only True \
  --optimize-ane True

# コンパイル
xcrun coremlc compile \
  models/coreml-encoder-large-v3-turbo.mlpackage models/

# 配置（ggmlモデルと同じディレクトリに置く）
mv models/coreml-encoder-large-v3-turbo.mlmodelc \
  ~/Library/Application\ Support/whisper/ggml-large-v3-turbo-encoder.mlmodelc
```

whisper.cppは、ggmlモデルファイルと同じディレクトリに `ggml-{model}-encoder.mlmodelc` があると自動的にCoreMLを使う。設定やフラグは不要。

### 3. アプリのビルド

Koeのbuild.shは `/tmp/whisper.cpp/build` にCoreMLビルドがあれば自動検出する:

```bash
WHISPER_COREML_BUILD="/tmp/whisper.cpp/build"
if [ -f "$WHISPER_COREML_BUILD/src/libwhisper.dylib" ] && \
   [ -f "$WHISPER_COREML_BUILD/src/libwhisper.coreml.dylib" ]; then
    echo "Using CoreML-enabled whisper.cpp build"
    USE_COREML=1
fi
```

## ハマったポイント

### dylib地獄：ggmlバージョン不一致

CoreMLビルドのwhisper.cppはggml 0.9.5を同梱するが、Homebrewのllama.cppはggml 0.9.7を要求する。同じアプリバンドルに両方入れると `Symbol not found: _ggml_build_forward_select` で即クラッシュ。

**解決**: whisper本体だけCoreMLビルドを使い、ggml系dyilibはHomebrew版（0.9.7）を共有する。

```bash
# whisper dylibs → CoreML build
cp /tmp/whisper.cpp/build/src/libwhisper.dylib Frameworks/
cp /tmp/whisper.cpp/build/src/libwhisper.coreml.dylib Frameworks/

# ggml dylibs → Homebrew (llama.cppと互換)
cp $(brew --prefix)/lib/libggml*.dylib Frameworks/
```

### rpath汚染

CoreMLビルドの `libwhisper.dylib` には `/tmp/whisper.cpp/build/ggml/src` がrpathとしてハードコードされる。これが残っていると、バンドル内のdylibより先に `/tmp` の古いdylibがロードされてクラッシュする。

```bash
# ビルドディレクトリのrpathを削除
for rp in $(otool -l Frameworks/libwhisper.dylib | \
  grep "path /tmp/" | awk '{print $2}'); do
    install_name_tool -delete_rpath "$rp" Frameworks/libwhisper.dylib
done
# @loader_pathに置換
install_name_tool -add_rpath "@loader_path" Frameworks/libwhisper.dylib
```

### struct layoutミスマッチ（前段の話）

実はCoreML以前に、もっと根本的な問題があった。SwiftからCのwhisper APIを呼ぶためのshim headerの`whisper_full_params`構造体が、インストール済みライブラリのバージョンと一致していなかった。

```c
// shim.hにあったが、v1.7.5には存在しないフィールド
bool carry_initial_prompt;  // v1.8.xで追加
struct whisper_vad_params vad_params;  // v1.8.xで追加
```

たった1つの `bool` フィールドのズレで、以降の全フィールドが8バイトずれる。`temperature`, `best_of`, `language` すべてがゴミ値を指すことになり、`whisper_full()` は成功（ret=0）を返すのに結果が0セグメント。

**教訓**: C構造体をFFIで使うときは、ヘッダーとライブラリのバージョンを1バイト単位で一致させること。`brew info whisper-cpp` でバージョン確認 → struct比較、をビルド前に必ずやる。

最終的にはC bridgeを作ってSwift側から構造体を渡さない設計にした:

```c
// whisper_bridge.c — C側で構造体を構築・設定
int whisper_bridge_transcribe(
    struct whisper_context *ctx,
    const float *samples, int n_samples,
    const char *language, const char *prompt,
    int n_threads, int best_of, ...
) {
    // C側でparams構築 → whisper_full()呼び出し
    struct whisper_full_params params = whisper_full_default_params(...);
    params.n_threads = n_threads;
    params.language = language;
    // ...
    return whisper_full(ctx, params, samples, n_samples);
}
```

## v1.7.5 → v1.8.3で速くなったか？

結論: **M2ではほぼ変わらない**。

```
v1.8.3のログ:
ggml_metal_device_init: tensor API disabled for pre-M5 and pre-A19 devices
```

v1.8.3のflash_attnやMetal最適化は、M5/A19以降のtensor APIに最適化されている。M2では`tensor API disabled`になるため恩恵がほぼない。速度差が出るのはCoreMLだけ。

| 設定 | 6秒音声の処理時間 |
|------|-----------------|
| v1.7.5 Metal | ~5.8秒 |
| v1.8.3 Metal | ~5.3秒 |
| v1.8.3 CoreML | **~1.5秒** |

## まとめ

- **CoreMLエンコーダ有効化で約5倍高速化**。Apple Silicon持ってるなら絶対やるべき
- whisper.cppのCoreMLサポートはエンコーダのみだが、処理時間の大部分がエンコーダなので効果絶大
- dylib依存関係とrpath管理が一番面倒。ggmlバージョン不一致に注意
- v1.8.3のMetal最適化はM5以降向け。M2/M3ではCoreMLが唯一の高速化手段

## v2.5.1 リリースノート

今回の変更は [v2.5.1](https://github.com/yukihamada/Koe-swift/releases/tag/v2.5.1) としてリリース済み。主な変更:

- **CoreML対応**: エンコーダをApple Neural Engineで実行、認識速度5倍
- **whisper.cpp v1.8.3**: Flash Attention、Metal最適化、VADサポート
- **左⌘/右⌘でIME切替**: 左コマンドで英語、右コマンドで日本語。macOS 13+で動作しなかった旧TIS APIをCGEventベースに修正
- **C bridge**: Swift-C間のstruct layoutミスマッチを根本解決
- **デフォルトモデル変更**: Large V3 Turbo Q5（高速・高精度・多言語対応）

Koeは無料・オープンソース: [github.com/yukihamada/Koe-swift](https://github.com/yukihamada/Koe-swift)

### インストール

ワンコマンドでインストールできます（whisper.cppの自動インストール + モデルダウンロード込み）:

```bash
curl -fsSL https://koe.elio.love/install.sh | bash
```

公式サイト: [koe.elio.love](https://koe.elio.love)
