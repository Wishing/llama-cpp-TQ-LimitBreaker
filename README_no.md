[ 🇺🇸 English](README.md) · [ 🇨🇳 简体中文](README_cn.md) · [ 🇯🇵 日本語](README_ja.md) · [ 🇰🇷 한국어](README_ko.md) · [ 🇻🇳 Tiếng Việt](README_vi.md) · [ 🇵🇭 Tagalog](README_tl.md) · [ 🇪🇸 Español](README_es.md) · [ 🇵🇹 Português](README_pt.md) · [ 🇮🇹 Italiano](README_it.md) · [ 🇩🇪 Deutsch](README_de.md) · [ 🇫🇷 Français](README_fr.md) · [ 🇸🇦 العربية](README_ar.md) · [ 🇮🇳 हिन्दी](README_hi.md) · [ 🇷🇺 Русский](README_ru.md) · [ 🇧🇩 বাংলা](README_bn.md) · [ 🇮🇱 עברית](README_he.md) · [ 🇵🇱 Polski](README_pl.md) · [ 🇨🇿 Čeština](README_cs.md) · [ 🇳🇱 Nederlands](README_nl.md) · [ 🇹🇷 Türkçe](README_tr.md) · [ 🇺🇦 Українська](README_uk.md) · [ 🇮🇩 Bahasa Indonesia](README_id.md) · [ 🇹🇭 ไทย](README_th.md) · [ 🇵🇰 اردو](README_ur.md) · [ 🇷🇴 Română](README_ro.md) · [ 🇸🇪 Svenska](README_sv.md) · [ 🇬🇷 Ελληνικά](README_el.md) · [ 🇭🇺 Magyar](README_hu.md) · [ 🇫🇮 Suomi](README_fi.md) · [ 🇩🇰 Dansk](README_da.md) · [ 🇳🇴 Norsk](README_no.md)

# 🚀 llama.cpp TurboQuant Limit Breaker (Klar for Gemma 4 & Qwopus)

> **Bryt de fysiske lovene for 24 GB VRAM: Kjør Gemma-4-31B / Qwopus-27B med 100 % GPU full offload på en enkelt RTX 3090, mens du opprettholder en massiv kontekst på 96K - 256K.**

## 🔍 Hvorfor dette prosjektet?

Per april 2026 står lokal distribusjon for AI-fellesskapet med åpen kildekode overfor et "dødvann":
1. **Offisiell forsinkelse**: Den offisielle `llama.cpp` mainline har ikke integrert TurboQuant-operatorer (`tbqp3_0`) i CUDA innfødt, noe som fører til at 30B+ modeller umiddelbart tømmer 24 GB VRAM under prefill av lang tekst.
2. **Kompatibilitetsproblemer med fellesskapsforker**: Eksisterende TurboQuant-grener i fellesskapet er trege til å oppdatere og er ikke kompatible med den nyeste SWA (Sliding Window) / ISWA hybrid cache-arkitekturen som ble introdusert i Gemma 4, noe som fører til `GGML_ASSERT` eller segmenteringsfeil.
3. **Dødssløyfe for agentskjema**: Når de pares med AI IDE-er som Claude Code, ZeroClaw eller Google Antigravity, utløser prefill-latensen for ultralange prompter enkelt timeout-forsøk fra klienten, noe som fører til at serveren havner i en dødssløyfe med høyt strømforbruk i tomgang med febrilske `cancel task`.

**Dette prosjektet gir en renset grensepatch (`llama_turboquant.patch`).** Den injiserer nøyaktig TurboQuants kjerneoperatorer for rotasjonskomprimering i den nyeste offisielle mainline-koden, løser manuelt C++-kildekonflikter og minnejusteringsfeil, og omgår perfekt krasjrøde linjer.

---

## 📊 Ekstreme ytelses-benchmarks

- **Maskinvare**: Enkelt NVIDIA GeForce RTX 3090 (24 GB VRAM)
- **Modeller**: `Gemma-4-31B-It-UD-IQ3_XXS.gguf` / `Qwopus3.5-27B-v3-Q4_K_M.gguf`
- **VRAM-optimalisering**: 
  - 98 304 (96K) kontekst initialisert, VRAM-bruk låst på **23 644 MiB / 24 576 MiB**.
  - KV Cache (K+V) komprimert fra >6 GB til **~1,48 GB**.
- **Genereringshastighet (Decode)**: Full `-ngl 99` akselerasjon, stabilt utdata på **~16,5 tokens/s**.

---

## 🛠️ Build- og injeksjonsguide

For å sikre at patchen blir brukt perfekt og for å unngå kompileringsfeil fra endringer i oppstrøms-API, **må du låse depotet ditt strengt til den testede spesifikke commit-versjonen**.

### 1. Forbered depotet og lås versjonen
```bash
git clone https://github.com/ggml-org/llama.cpp.git
cd llama.cpp

# MÅ låses til denne sikre versjonen (Build b8728)
git checkout 5e9c63546

# Bruk patch og bygg
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
