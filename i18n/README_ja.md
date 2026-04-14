[ 🇺🇸 English](../README.md) · [ 🇨🇳 简体中文](README_cn.md) · [ 🇯🇵 日本語](README_ja.md) · [ 🇰🇷 한국어](README_ko.md) · [ 🇻🇳 Tiếng Việt](README_vi.md) · [ 🇵🇭 Tagalog](README_tl.md) · [ 🇪🇸 Español](README_es.md) · [ 🇵🇹 Português](README_pt.md) · [ 🇮🇹 Italiano](README_it.md) · [ 🇩🇪 Deutsch](README_de.md) · [ 🇫🇷 Français](README_fr.md) · [ 🇸🇦 العربية](README_ar.md) · [ 🇮🇳 हिन्दी](README_hi.md) · [ 🇷🇺 Русский](README_ru.md) · [ 🇧🇩 বাংলা](README_bn.md) · [ 🇮🇱 עברית](README_he.md) · [ 🇵🇱 Polski](README_pl.md) · [ 🇨🇿 Čeština](README_cs.md) · [ 🇳🇱 Nederlands](README_nl.md) · [ 🇹🇷 Türkçe](README_tr.md) · [ 🇺🇦 Українська](README_uk.md) · [ 🇮🇩 Bahasa Indonesia](README_id.md) · [ 🇹🇭 ไทย](README_th.md) · [ 🇵🇰 اردو](README_ur.md) · [ 🇷🇴 Română](README_ro.md) · [ 🇸🇪 Svenska](README_sv.md) · [ 🇬🇷 Ελληνικά](README_el.md) · [ 🇭🇺 Magyar](README_hu.md) · [ 🇫🇮 Suomi](README_fi.md) · [ 🇩🇰 Dansk](README_da.md) · [ 🇳🇴 Norsk](README_no.md)

# 🚀 llama.cpp TurboQuant Limit Breaker (Gemma 4 & Qwopus 対応)

> **24GB VRAM の物理法則を打破：単一の RTX 3090 で Gemma-4-31B / Qwopus-27B を 100% GPU フルオフロードで実行し、96K - 256K の大規模なコンテキストを維持。**

## 🔍 なぜこのプロジェクトが必要なのか？

2026年4月現在、オープンソース AI コミュニティのローカルデプロイメントは「デッドロック」に直面しています：
1. **公式の遅れ**: 公式の `llama.cpp` メインラインは TurboQuant 演算子 (`tbqp3_0`) を CUDA にネイティブ統合しておらず、30B 以上のモデルでは長いテキストのプリフィル中に 24GB VRAM を瞬時に使い果たしてしまいます。
2. **コミュニティフォークの互換性問題**: 既存のコミュニティ TurboQuant ブランチは更新が遅く、Gemma 4 で導入された最新の SWA (スライディングウィンドウ) / ISWA ハイブリッドキャッシュアーキテクチャと互換性がなく、`GGML_ASSERT` やセグメンテーションフォールトを引き起こします。
3. **エージェントスケジューリングのデスループ**: Claude Code、ZeroClaw、Google Antigravity などの AI IDE と組み合わせると、超長文プロンプトのプリフィル遅延によりクライアントのタイムアウト再試行が誘発され、サーバーが「タスクキャンセル」を繰り返す高電力アイドルのデスループに陥ります。

**このプロジェクトは、精製されたリミットパッチ (`llama_turboquant.patch`) を提供します。** 最新の公式メインラインコードにコアとなる TurboQuant 回転圧縮演算子を正確に注入し、C++ ソースの競合やメモリ配置エラーを手動で解決することで、クラッシュの境界線を完璧に回避します。

---

## 📊 極限性能ベンチマーク

- **ハードウェア**: 単一 NVIDIA GeForce RTX 3090 (24GB VRAM)
- **モデル**: `Gemma-4-31B-It-UD-IQ3_XXS.gguf` / `Qwopus3.5-27B-v3-Q4_K_M.gguf`
- **VRAM 最適化**: 
  - 98,304 (96K) コンテキスト初期化時、VRAM 使用量を **23,644 MiB / 24,576 MiB** に固定。
  - KV キャッシュ (K+V) を 6GB 以上から **約 1.48 GB** に圧縮。
- **生成速度 (Decode)**: フル `-ngl 99` 加速により、**約 16.5 tokens/s** で安定出力。

---

## 🛠️ ビルド＆インジェクションガイド

パッチを完璧に適用し、アップストリームの API 変更によるコンパイルエラーを避けるため、**必ずテスト済みの特定のコミットバージョンにリポジトリをロックしてください**。

### 1. リポジトリの準備とバージョンのロック
```bash
git clone https://github.com/ggml-org/llama.cpp.git
cd llama.cpp

# 必ずこの安全なバージョンにロックしてください (Build b8728)
git checkout 5e9c63546

# パッチの適用とビルド
git apply llama_turboquant.patch
cmake -B build -DGGML_CUDA=ON -DGGML_CUDA_F16=ON
cmake --build build --config Release -j $(nproc) --target llama-server
./build/bin/llama-server \
    -m /path/to/gemma-4-31B-it-UD-IQ3_XXS.gguf \
    -c 98304 \
    -b 1024 -ub 512 \
    -ngl 99 \
    -fa \
    -ctk tbqp3_0 -ctv tbq3_0 \
    --parallel 1 \
    --no-prompt-cache \
    --timeout 3600 \
    --port 1337
```
