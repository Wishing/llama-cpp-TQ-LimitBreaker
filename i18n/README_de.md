[ 🇺🇸 English](../README.md) · [ 🇨🇳 简体中文](README_cn.md) · [ 🇯🇵 日本語](README_ja.md) · [ 🇰🇷 한국어](README_ko.md) · [ 🇻🇳 Tiếng Việt](README_vi.md) · [ 🇵🇭 Tagalog](README_tl.md) · [ 🇪🇸 Español](README_es.md) · [ 🇵🇹 Português](README_pt.md) · [ 🇮🇹 Italiano](README_it.md) · [ 🇩🇪 Deutsch](README_de.md) · [ 🇫🇷 Français](README_fr.md) · [ 🇸🇦 العربية](README_ar.md) · [ 🇮🇳 हिन्दी](README_hi.md) · [ 🇷🇺 Русский](README_ru.md) · [ 🇧🇩 বাংলা](README_bn.md) · [ 🇮🇱 עברית](README_he.md) · [ 🇵🇱 Polski](README_pl.md) · [ 🇨🇿 Čeština](README_cs.md) · [ 🇳🇱 Nederlands](README_nl.md) · [ 🇹🇷 Türkçe](README_tr.md) · [ 🇺🇦 Українська](README_uk.md) · [ 🇮🇩 Bahasa Indonesia](README_id.md) · [ 🇹🇭 ไทย](README_th.md) · [ 🇵🇰 اردو](README_ur.md) · [ 🇷🇴 Română](README_ro.md) · [ 🇸🇪 Svenska](README_sv.md) · [ 🇬🇷 Ελληνικά](README_el.md) · [ 🇭🇺 Magyar](README_hu.md) · [ 🇫🇮 Suomi](README_fi.md) · [ 🇩🇰 Dansk](README_da.md) · [ 🇳🇴 Norsk](README_no.md)

# 🚀 llama.cpp TurboQuant Limit Breaker (Bereit für Gemma 4 & Qwopus)

> **Die physikalischen Gesetze von 24GB VRAM brechen: Führen Sie Gemma-4-31B / Qwopus-27B mit 100% GPU-Full-Offload auf einer einzigen RTX 3090 aus, während Sie einen massiven Kontext von 96K - 256K beibehalten.**

## 🔍 Warum dieses Projekt?

Stand April 2026 steht die lokale Bereitstellung der Open-Source-AI-Community vor einem „Deadlock“:
1. **Offizielle Verzögerung**: Die offizielle `llama.cpp`-Mainline hat TurboQuant-Operatoren (`tbqp3_0`) nicht nativ in CUDA integriert, was dazu führt, dass 30B+-Modelle während des Prefills von langen Texten sofort 24GB VRAM verbrauchen.
2. **Kompatibilitätsprobleme von Community-Forks**: Bestehende TurboQuant-Branches der Community werden nur langsam aktualisiert und sind nicht mit der neuesten SWA (Sliding Window) / ISWA-Hybrid-Cache-Architektur kompatibel, die in Gemma 4 eingeführt wurde, was zu `GGML_ASSERT` oder Segmentierungsfehlern führt.
3. **Agent-Scheduling-Todesschleife**: In Verbindung mit AI-IDEs wie Claude Code, ZeroClaw oder Google Antigravity löst die Prefill-Latenz von ultralangen Prompts leicht Zeitüberschreitungs-Wiederholungsversuche des Clients aus, wodurch der Server in eine stromfressende Leerlauf-Todesschleife von hektischen `cancel task`-Vorgängen gerät.

**Dieses Projekt bietet einen gereinigten Limit-Patch (`llama_turboquant.patch`).** Er injiziert die Kern-TurboQuant-Rotationskompressionsoperatoren präzise in den neuesten offiziellen Mainline-Code, wobei C++-Quellkonflikte und Speicherausrichtungsfehler manuell gelöst werden, um die Absturzgrenzen perfekt zu umgehen.

---

## 📊 Extreme Performance-Benchmarks

- **Hardware**: Einzelne NVIDIA GeForce RTX 3090 (24GB VRAM)
- **Modelle**: `Gemma-4-31B-It-UD-IQ3_XXS.gguf` / `Qwopus3.5-27B-v3-Q4_K_M.gguf`
- **VRAM-Optimierung**: 
  - 98.304 (96K) Kontext initialisiert, VRAM-Nutzung bei **23.644 MiB / 24.576 MiB** gesperrt.
  - KV Cache (K+V) von >6GB auf **~1,48 GB** komprimiert.
- **Generationsgeschwindigkeit (Decode)**: Volle `-ngl 99` Beschleunigung, stabile Ausgabe bei **~16,5 tokens/s**.

---

## 🛠️ Build- & Injektionsanleitung

Um sicherzustellen, dass der Patch perfekt angewendet wird und um Kompilierungsfehler durch Upstream-API-Änderungen zu vermeiden, **sperren Sie Ihr Repository bitte strikt auf die getestete spezifische Commit-Version**.

### 1. Repository vorbereiten & Version sperren
```bash
git clone https://github.com/ggml-org/llama.cpp.git
cd llama.cpp

# MUSS auf diese sichere Version gesperrt werden (Build b8728)
git checkout 5e9c63546

# Patch anwenden und builden
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
