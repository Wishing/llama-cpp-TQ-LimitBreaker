[ 🇺🇸 English](README.md) · [ 🇨🇳 简体中文](README_cn.md) · [ 🇯🇵 日本語](README_ja.md) · [ 🇰🇷 한국어](README_ko.md) · [ 🇻🇳 Tiếng Việt](README_vi.md) · [ 🇵🇭 Tagalog](README_tl.md) · [ 🇪🇸 Español](README_es.md) · [ 🇵🇹 Português](README_pt.md) · [ 🇮🇹 Italiano](README_it.md) · [ 🇩🇪 Deutsch](README_de.md) · [ 🇫🇷 Français](README_fr.md) · [ 🇸🇦 العربية](README_ar.md) · [ 🇮🇳 हिन्दी](README_hi.md) · [ 🇷🇺 Русский](README_ru.md) · [ 🇧🇩 বাংলা](README_bn.md) · [ 🇮🇱 עברית](README_he.md) · [ 🇵🇱 Polski](README_pl.md) · [ 🇨🇿 Čeština](README_cs.md) · [ 🇳🇱 Nederlands](README_nl.md) · [ 🇹🇷 Türkçe](README_tr.md) · [ 🇺🇦 Українська](README_uk.md) · [ 🇮🇩 Bahasa Indonesia](README_id.md) · [ 🇹🇭 ไทย](README_th.md) · [ 🇵🇰 اردو](README_ur.md) · [ 🇷🇴 Română](README_ro.md) · [ 🇸🇪 Svenska](README_sv.md) · [ 🇬🇷 Ελληνικά](README_el.md) · [ 🇭🇺 Magyar](README_hu.md) · [ 🇫🇮 Suomi](README_fi.md) · [ 🇩🇰 Dansk](README_da.md) · [ 🇳🇴 Norsk](README_no.md)

# 🚀 llama.cpp TurboQuant Limit Breaker (Pronto per Gemma 4 e Qwopus)

> **Rompere le leggi della fisica di 24GB VRAM: Esegui Gemma-4-31B / Qwopus-27B con scaricamento GPU completo al 100% su una singola RTX 3090, mantenendo un contesto massivo di 96K - 256K.**

## 🔍 Perché questo progetto?

A partire da aprile 2026, la distribuzione locale della comunità AI open source si trova di fronte a un "vicolo cieco":
1. **Ritardo Ufficiale**: Il ramo principale ufficiale di `llama.cpp` non ha integrato nativamente gli operatori TurboQuant (`tbqp3_0`) in CUDA, causando l'esaurimento istantaneo di 24GB di VRAM per i modelli 30B+ durante il prefill di testi lunghi.
2. **Problemi di Compatibilità dei Fork della Comunità**: I rami TurboQuant della comunità esistenti sono lenti ad aggiornarsi e non sono compatibili con l'ultima architettura cache ibrida SWA (Sliding Window) / ISWA introdotta in Gemma 4, portando a `GGML_ASSERT` o errori di segmentazione.
3. **Loop della Morte dello Scheduling degli Agenti**: Se abbinato a IDE AI come Claude Code, ZeroClaw o Google Antigravity, la latenza di prefill di prompt ultra-lunghi attiva facilmente i tentativi di timeout del client, facendo cadere il server in un loop della morte in idle ad alta potenza di frenetici `cancel task`.

**Questo progetto fornisce una patch di limite purificata (`llama_turboquant.patch`).** Inietta con precisione i core operatori di compressione rotazione TurboQuant nel codice del ramo principale ufficiale più recente, risolvendo manualmente i conflitti dei sorgenti C++ e gli errori di allineamento della memoria, aggirando perfettamente le linee rosse dei crash.

---

## 📊 Benchmark di Prestazioni Estreme

- **Hardware**: Singola NVIDIA GeForce RTX 3090 (24GB VRAM)
- **Modelli**: `Gemma-4-31B-It-UD-IQ3_XXS.gguf` / `Qwopus3.5-27B-v3-Q4_K_M.gguf`
- **Ottimizzazione VRAM**: 
  - Contesto di 98.304 (96K) inizializzato, utilizzo VRAM bloccato a **23.644 MiB / 24.576 MiB**.
  - KV Cache (K+V) compressa da >6GB a **~1.48 GB**.
- **Velocità di Generazione (Decode)**: Accelerazione `-ngl 99` completa, output stabile a **~16.5 tokens/s**.

---

## 🛠️ Guida alla Build e all'Iniezione

Per garantire che la patch venga applicata perfettamente e per evitare errori di compilazione dovuti a modifiche delle API a monte, **si prega di bloccare rigorosamente il repository alla versione specifica del commit testato**.

### 1. Preparazione del Repository e Blocco della Versione
```bash
git clone https://github.com/ggml-org/llama.cpp.git
cd llama.cpp

# DEVE bloccare a questa versione sicura (Build b8728)
git checkout 5e9c63546

# Applica la patch e build
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
