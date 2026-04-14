[ 🇺🇸 English](README.md) · [ 🇨🇳 简体中文](README_cn.md) · [ 🇯🇵 日本語](README_ja.md) · [ 🇰🇷 한국어](README_ko.md) · [ 🇻🇳 Tiếng Việt](README_vi.md) · [ 🇵🇭 Tagalog](README_tl.md) · [ 🇪🇸 Español](README_es.md) · [ 🇵🇹 Português](README_pt.md) · [ 🇮🇹 Italiano](README_it.md) · [ 🇩🇪 Deutsch](README_de.md) · [ 🇫🇷 Français](README_fr.md) · [ 🇸🇦 العربية](README_ar.md) · [ 🇮🇳 हिन्दी](README_hi.md) · [ 🇷🇺 Русский](README_ru.md) · [ 🇧🇩 বাংলা](README_bn.md) · [ 🇮🇱 עברית](README_he.md) · [ 🇵🇱 Polski](README_pl.md) · [ 🇨🇿 Čeština](README_cs.md) · [ 🇳🇱 Nederlands](README_nl.md) · [ 🇹🇷 Türkçe](README_tr.md) · [ 🇺🇦 Українська](README_uk.md) · [ 🇮🇩 Bahasa Indonesia](README_id.md) · [ 🇹🇭 ไทย](README_th.md) · [ 🇵🇰 اردو](README_ur.md) · [ 🇷🇴 Română](README_ro.md) · [ 🇸🇪 Svenska](README_sv.md) · [ 🇬🇷 Ελληνικά](README_el.md) · [ 🇭🇺 Magyar](README_hu.md) · [ 🇫🇮 Suomi](README_fi.md) · [ 🇩🇰 Dansk](README_da.md) · [ 🇳🇴 Norsk](README_no.md)

# 🚀 llama.cpp TurboQuant Limit Breaker (Gata pentru Gemma 4 & Qwopus)

> **Depășirea legilor fizicii pentru 24GB VRAM: Rulați Gemma-4-31B / Qwopus-27B cu 100% GPU full offload pe un singur RTX 3090, menținând în același timp un context masiv de 96K - 256K.**

## 🔍 De ce acest proiect?

Începând cu aprilie 2026, implementarea locală a comunității AI open-source se confruntă cu un „blocaj”:
1. **Întârziere Oficială**: Linia principală (mainline) oficială a `llama.cpp` nu a integrat nativ operatorii TurboQuant (`tbqp3_0`) în CUDA, ceea ce face ca modelele de peste 30B să consume instantaneu 24GB VRAM în timpul prefill-ului textelor lungi.
2. **Probleme de compatibilitate ale fork-urilor comunității**: Ramurile TurboQuant existente în comunitate se actualizează greu și nu sunt compatibile cu cea mai recentă arhitectură de cache hibrid SWA (Sliding Window) / ISWA introdusă în Gemma 4, ducând la erori `GGML_ASSERT` sau eșecuri de segmentare (segmentation faults).
3. **Agent Scheduling Death Loop**: Atunci când este utilizat cu AI IDE-uri precum Claude Code, ZeroClaw sau Google Antigravity, latența de prefill a prompt-urilor ultra-lungi declanșează cu ușurință reîncercări din cauza timeout-ului clientului, făcând ca serverul să cadă într-un cerc vicios de consum ridicat la ralanti cu anulări frenetice de sarcini (`cancel task`).

**Acest proiect oferă un patch de limită purificat (`llama_turboquant.patch`).** Acesta injectează cu precizie operatorii de compresie prin rotație TurboQuant în cel mai recent cod oficial, rezolvând manual conflictele de sursă C++ și erorile de aliniere a memoriei, ocolind perfect pragurile de crash.

---

## 📊 Benchmark-uri de performanță extremă

- **Hardware**: O singură placă NVIDIA GeForce RTX 3090 (24GB VRAM)
- **Modele**: `Gemma-4-31B-It-UD-IQ3_XXS.gguf` / `Qwopus3.5-27B-v3-Q4_K_M.gguf`
- **Optimizare VRAM**: 
  - Context de 98.304 (96K) inițializat, utilizarea VRAM blocată la **23.644 MiB / 24.576 MiB**.
  - KV Cache (K+V) comprimat de la >6GB la **~1.48 GB**.
- **Viteza de generare (Decode)**: Accelerare completă `-ngl 99`, ieșire stabilă la **~16.5 tokens/s**.

---

## 🛠️ Ghid de Build și Injecție

Pentru a vă asigura că patch-ul este aplicat perfect și pentru a evita erorile de compilare cauzate de modificările API-ului upstream, **vă rugăm să blocați strict depozitul la versiunea specifică de commit testată**.

### 1. Pregătirea Depozitului și Blocarea Versiunii
```bash
git clone https://github.com/ggml-org/llama.cpp.git
cd llama.cpp

# TREBUIE să blocați la această versiune sigură (Build b8728)
git checkout 5e9c63546

# Aplicați patch-ul și construiți
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
