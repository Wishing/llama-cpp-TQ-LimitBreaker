[ 🇺🇸 English](README.md) · [ 🇨🇳 简体中文](README_cn.md) · [ 🇯🇵 日本語](README_ja.md) · [ 🇰🇷 한국어](README_ko.md) · [ 🇻🇳 Tiếng Việt](README_vi.md) · [ 🇵🇭 Tagalog](README_tl.md) · [ 🇪🇸 Español](README_es.md) · [ 🇵🇹 Português](README_pt.md) · [ 🇮🇹 Italiano](README_it.md) · [ 🇩🇪 Deutsch](README_de.md) · [ 🇫🇷 Français](README_fr.md) · [ 🇸🇦 العربية](README_ar.md) · [ 🇮🇳 हिन्दी](README_hi.md) · [ 🇷🇺 Русский](README_ru.md) · [ 🇧🇩 বাংলা](README_bn.md) · [ 🇮🇱 עברית](README_he.md) · [ 🇵🇱 Polski](README_pl.md) · [ 🇨🇿 Čeština](README_cs.md) · [ 🇳🇱 Nederlands](README_nl.md) · [ 🇹🇷 Türkçe](README_tr.md) · [ 🇺🇦 Українська](README_uk.md) · [ 🇮🇩 Bahasa Indonesia](README_id.md) · [ 🇹🇭 ไทย](README_th.md) · [ 🇵🇰 اردو](README_ur.md) · [ 🇷🇴 Română](README_ro.md) · [ 🇸🇪 Svenska](README_sv.md) · [ 🇬🇷 Ελληνικά](README_el.md) · [ 🇭🇺 Magyar](README_hu.md) · [ 🇫🇮 Suomi](README_fi.md) · [ 🇩🇰 Dansk](README_da.md) · [ 🇳🇴 Norsk](README_no.md)

# 🚀 llama.cpp TurboQuant Limit Breaker (Redo för Gemma 4 & Qwopus)

> **Bryt de fysiska lagarna för 24 GB VRAM: Kör Gemma-4-31B / Qwopus-27B med 100 % GPU full offload på en enda RTX 3090, samtidigt som du bibehåller en massiv kontext på 96K - 256K.**

## 🔍 Varför detta projekt?

Från och med april 2026 står den lokala distributionen för AI-communityn med öppen källkod inför ett "dödläge":
1. **Officiell eftersläpning**: Den officiella `llama.cpp`-mainline har inte integrerat TurboQuant-operatorer (`tbqp3_0`) i CUDA nativt, vilket gör att modeller på 30B+ omedelbart tömmer 24 GB VRAM under prefill av lång text.
2. **Kompatibilitetsproblem med community-forkar**: Befintliga TurboQuant-grenar i communityn är långsamma att uppdatera och är inte kompatibla med den senaste SWA (Sliding Window) / ISWA-hybridcachearkitekturen som introducerades i Gemma 4, vilket leder till `GGML_ASSERT` eller segmenteringsfel.
3. **Dödsslinga för agentschemaläggning**: När de paras ihop med AI IDE:er som Claude Code, ZeroClaw eller Google Antigravity, utlöser prefill-latensen för ultralånga prompter enkelt klient-timeout-försök, vilket gör att servern hamnar i en dödsslinga med hög effekt och febrila `cancel task`.

**Detta projekt tillhandahåller en renad gränspatch (`llama_turboquant.patch`).** Den injicerar exakt TurboQuants kärnoperatorer för rotationskomprimering i den senaste officiella mainline-koden, löser manuellt C++-källkonflikter och minnesjusteringsfel, och kringgår perfekt kraschgränserna.

---

## 📊 Benchmarks för extrem prestanda

- **Hårdvara**: En enda NVIDIA GeForce RTX 3090 (24 GB VRAM)
- **Modeller**: `Gemma-4-31B-It-UD-IQ3_XXS.gguf` / `Qwopus3.5-27B-v3-Q4_K_M.gguf`
- **VRAM-optimering**: 
  - 98 304 (96K) kontext initierad, VRAM-användning låst till **23 644 MiB / 24 576 MiB**.
  - KV Cache (K+V) komprimerad från >6 GB till **~1,48 GB**.
- **Generationshastighet (Decode)**: Full `-ngl 99`-acceleration, stabil utmatning på **~16,5 tokens/s**.

---

## 🛠️ Build- & injektionsguide

För att säkerställa att patchen appliceras perfekt och för att undvika kompileringsfel från ändringar i uppströms-API, **vänligen lås ditt arkiv strikt till den testade specifika commit-versionen**.

### 1. Förbered arkivet och lås versionen
```bash
git clone https://github.com/ggml-org/llama.cpp.git
cd llama.cpp

# MÅSTE låsas till denna säkra version (Build b8728)
git checkout 5e9c63546

# Applicera patch och bygg
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
