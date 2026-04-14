[ 🇺🇸 English](README.md) · [ 🇨🇳 简体中文](README_cn.md) · [ 🇯🇵 日本語](README_ja.md) · [ 🇰🇷 한국어](README_ko.md) · [ 🇻🇳 Tiếng Việt](README_vi.md) · [ 🇵🇭 Tagalog](README_tl.md) · [ 🇪🇸 Español](README_es.md) · [ 🇵🇹 Português](README_pt.md) · [ 🇮🇹 Italiano](README_it.md) · [ 🇩🇪 Deutsch](README_de.md) · [ 🇫🇷 Français](README_fr.md) · [ 🇸🇦 العربية](README_ar.md) · [ 🇮🇳 हिन्दी](README_hi.md) · [ 🇷🇺 Русский](README_ru.md) · [ 🇧🇩 বাংলা](README_bn.md) · [ 🇮🇱 עברית](README_he.md) · [ 🇵🇱 Polski](README_pl.md) · [ 🇨🇿 Čeština](README_cs.md) · [ 🇳🇱 Nederlands](README_nl.md) · [ 🇹🇷 Türkçe](README_tr.md) · [ 🇺🇦 Українська](README_uk.md) · [ 🇮🇩 Bahasa Indonesia](README_id.md) · [ 🇹🇭 ไทย](README_th.md) · [ 🇵🇰 اردو](README_ur.md) · [ 🇷🇴 Română](README_ro.md) · [ 🇸🇪 Svenska](README_sv.md) · [ 🇬🇷 Ελληνικά](README_el.md) · [ 🇭🇺 Magyar](README_hu.md) · [ 🇫🇮 Suomi](README_fi.md) · [ 🇩🇰 Dansk](README_da.md) · [ 🇳🇴 Norsk](README_no.md)

# 🚀 llama.cpp TurboQuant Limit Breaker (Gemma 4 és Qwopus készen áll)

> **A 24 GB VRAM fizikai törvényeinek áttörése: Futtassa a Gemma-4-31B / Qwopus-27B modellt 100% GPU-s offloaddal egyetlen RTX 3090-esen, miközben 96K - 256K hatalmas kontextust tart fenn.**

## 🔍 Miért ez a projekt?

2026 áprilisától a nyílt forráskódú AI közösség helyi telepítése „patthelyzettel” néz szembe:
1. **Hivatalos lemaradás**: A hivatalos `llama.cpp` mainline nem integrálta natívan a TurboQuant operátorokat (`tbqp3_0`) a CUDA-ba, ami miatt a 30B+ modellek azonnal kimerítik a 24 GB VRAM-ot a hosszú szövegek előtöltése (prefill) során.
2. **Közösségi fork kompatibilitási problémák**: A meglévő közösségi TurboQuant ágak lassan frissülnek, és nem kompatibilisek a Gemma 4-ben bevezetett legújabb SWA (Sliding Window) / ISWA hibrid cache architektúrával, ami `GGML_ASSERT` hibához vagy szegmentálási hibához vezet.
3. **Agent Scheduling Death Loop**: Ha olyan AI IDE-kkel párosítják, mint a Claude Code, a ZeroClaw vagy a Google Antigravity, az ultra-hosszú promptok előtöltési késleltetése könnyen kiváltja a kliens időtúllépési újrapróbálkozásait, aminek következtében a szerver a feladatok eszeveszett törlésének (`cancel task`) nagy teljesítményű üresjárati halálciklusába esik.

**Ez a projekt egy tisztított limit-javítást (`llama_turboquant.patch`) kínál.** Pontosan beinjektálja az alapvető TurboQuant rotációs tömörítési operátorokat a legújabb hivatalos mainline kódba, manuálisan feloldva a C++ forráskonfliktusokat és a memória-igazítási hibákat, tökéletesen megkerülve az összeomlási vörös vonalakat.

---

## 📊 Extrém teljesítmény-összehasonlítások

- **Hardver**: Egyetlen NVIDIA GeForce RTX 3090 (24 GB VRAM)
- **Modellek**: `Gemma-4-31B-It-UD-IQ3_XXS.gguf` / `Qwopus3.5-27B-v3-Q4_K_M.gguf`
- **VRAM optimalizálás**: 
  - 98 304 (96K) kontextus inicializálva, a VRAM használat **23 644 MiB / 24 576 MiB** értéken rögzítve.
  - A KV Cache (K+V) >6 GB-ról **~1,48 GB**-ra tömörítve.
- **Generálási sebesség (Decode)**: Teljes `-ngl 99` gyorsítás, stabil kimenet **~16,5 tokens/s** sebességgel.

---

## 🛠️ Build és injektálási útmutató

A javítás tökéletes alkalmazása és az upstream API változások miatti fordítási hibák elkerülése érdekében **kérjük, szigorúan rögzítse a tárolót a tesztelt specifikus commit verzióhoz**.

### 1. Tároló előkészítése és verzió rögzítése
```bash
git clone https://github.com/ggml-org/llama.cpp.git
cd llama.cpp

# Kötelező ehhez a biztonságos verzióhoz rögzíteni (Build b8728)
git checkout 5e9c63546

# Javítás alkalmazása és build
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
