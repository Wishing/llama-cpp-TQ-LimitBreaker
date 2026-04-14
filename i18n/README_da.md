[ 🇺🇸 English](../README.md) · [ 🇨🇳 简体中文](README_cn.md) · [ 🇯🇵 日本語](README_ja.md) · [ 🇰🇷 한국어](README_ko.md) · [ 🇻🇳 Tiếng Việt](README_vi.md) · [ 🇵🇭 Tagalog](README_tl.md) · [ 🇪🇸 Español](README_es.md) · [ 🇵🇹 Português](README_pt.md) · [ 🇮🇹 Italiano](README_it.md) · [ 🇩🇪 Deutsch](README_de.md) · [ 🇫🇷 Français](README_fr.md) · [ 🇸🇦 العربية](README_ar.md) · [ 🇮🇳 हिन्दी](README_hi.md) · [ 🇷🇺 Русский](README_ru.md) · [ 🇧🇩 বাংলা](README_bn.md) · [ 🇮🇱 עברית](README_he.md) · [ 🇵🇱 Polski](README_pl.md) · [ 🇨🇿 Čeština](README_cs.md) · [ 🇳🇱 Nederlands](README_nl.md) · [ 🇹🇷 Türkçe](README_tr.md) · [ 🇺🇦 Українська](README_uk.md) · [ 🇮🇩 Bahasa Indonesia](README_id.md) · [ 🇹🇭 ไทย](README_th.md) · [ 🇵🇰 اردو](README_ur.md) · [ 🇷🇴 Română](README_ro.md) · [ 🇸🇪 Svenska](README_sv.md) · [ 🇬🇷 Ελληνικά](README_el.md) · [ 🇭🇺 Magyar](README_hu.md) · [ 🇫🇮 Suomi](README_fi.md) · [ 🇩🇰 Dansk](README_da.md) · [ 🇳🇴 Norsk](README_no.md)

# 🚀 llama.cpp TurboQuant Limit Breaker (Klar til Gemma 4 & Qwopus)

> **Nedbrydning af de fysiske love for 24GB VRAM: Kør Gemma-4-31B / Qwopus-27B med 100% GPU full offload på en enkelt RTX 3090, mens du opretholder en massiv kontekst på 96K - 256K.**

## 🔍 Hvorfor dette projekt?

Pr. april 2026 står lokal implementering i open-source AI-community'et over for et "dødvande":
1. **Officiel forsinkelse**: Den officielle `llama.cpp` mainline har ikke integreret TurboQuant-operatorer (`tbqp3_0`) i CUDA indfødt, hvilket får 30B+ modeller til øjeblikkeligt at udtømme 24GB VRAM under prefill af lang tekst.
2. **Kompatibilitetsproblemer med community-forks**: Eksisterende community TurboQuant-branches er langsomme til at opdatere og er ikke kompatible med den nyeste SWA (Sliding Window) / ISWA hybrid cache-arkitektur introduceret i Gemma 4, hvilket fører til `GGML_ASSERT` eller segmenteringsfejl.
3. **Agent Scheduling Death Loop**: Når det parres med AI IDE'er som Claude Code, ZeroClaw eller Google Antigravity, udløser prefill-latensen for ultralange prompts nemt klient-timeout-forsøg, hvilket får serveren til at falde ind i et strømslugende standby-dødloop af febrilske `cancel task`.

**Dette projekt leverer et renset limit-patch (`llama_turboquant.patch`).** Det injicerer præcist kerne-TurboQuant-rotationskomprimeringsoperatorerne i den nyeste officielle mainline-kode, løser manuelt C++ kildekonflikter og hukommelsesjusteringsfejl, og omgår perfekt crash-røde linjer.

---

## 📊 Ekstreme ydelses-benchmarks

- **Hardware**: Enkelt NVIDIA GeForce RTX 3090 (24GB VRAM)
- **Modeller**: `Gemma-4-31B-It-UD-IQ3_XXS.gguf` / `Qwopus3.5-27B-v3-Q4_K_M.gguf`
- **VRAM-optimering**: 
  - 98.304 (96K) kontekst initialiseret, VRAM-forbrug låst på **23.644 MiB / 24.576 MiB**.
  - KV Cache (K+V) komprimeret fra >6GB til **~1.48 GB**.
- **Generationshastighed (Decode)**: Fuld `-ngl 99` acceleration, stabilt output på **~16,5 tokens/s**.

---

## 🛠️ Build- & injektionsvejledning

For at sikre, at patchet anvendes perfekt og for at undgå kompileringsfejl fra upstream API-ændringer, **skal du strengt låse dit repository til den testede specifikke commit-version**.

### 1. Forbered repository & lås version
```bash
git clone https://github.com/ggml-org/llama.cpp.git
cd llama.cpp

# SKAL låses til denne sikre version (Build b8728)
git checkout 5e9c63546

# Anvend patch og build
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
