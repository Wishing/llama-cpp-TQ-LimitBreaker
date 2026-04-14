[ 🇺🇸 English](README.md) · [ 🇨🇳 简体中文](README_cn.md) · [ 🇯🇵 日本語](README_ja.md) · [ 🇰🇷 한국어](README_ko.md) · [ 🇻🇳 Tiếng Việt](README_vi.md) · [ 🇵🇭 Tagalog](README_tl.md) · [ 🇪🇸 Español](README_es.md) · [ 🇵🇹 Português](README_pt.md) · [ 🇮🇹 Italiano](README_it.md) · [ 🇩🇪 Deutsch](README_de.md) · [ 🇫🇷 Français](README_fr.md) · [ 🇸🇦 العربية](README_ar.md) · [ 🇮🇳 हिन्दी](README_hi.md) · [ 🇷🇺 Русский](README_ru.md) · [ 🇧🇩 বাংলা](README_bn.md) · [ 🇮🇱 עברית](README_he.md) · [ 🇵🇱 Polski](README_pl.md) · [ 🇨🇿 Čeština](README_cs.md) · [ 🇳🇱 Nederlands](README_nl.md) · [ 🇹🇷 Türkçe](README_tr.md) · [ 🇺🇦 Українська](README_uk.md) · [ 🇮🇩 Bahasa Indonesia](README_id.md) · [ 🇹🇭 ไทย](README_th.md) · [ 🇵🇰 اردو](README_ur.md) · [ 🇷🇴 Română](README_ro.md) · [ 🇸🇪 Svenska](README_sv.md) · [ 🇬🇷 Ελληνικά](README_el.md) · [ 🇭🇺 Magyar](README_hu.md) · [ 🇫🇮 Suomi](README_fi.md) · [ 🇩🇰 Dansk](README_da.md) · [ 🇳🇴 Norsk](README_no.md)

# 🚀 llama.cpp TurboQuant Limit Breaker (Gereed voor Gemma 4 & Qwopus)

> **De natuurwetten van 24GB VRAM doorbreken: Draai Gemma-4-31B / Qwopus-27B met 100% GPU full offload auf een enkele RTX 3090, terwijl een gigantische context van 96K - 256K behouden blijft.**

## 🔍 Waarom dit project?

Vanaf april 2026 staat de lokale inzet van de open-source AI-gemeenschap voor een "impasse":
1. **Officiële vertraging**: De officiële `llama.cpp` mainline heeft TurboQuant-operators (`tbqp3_0`) niet standaard in CUDA geïntegreerd, waardoor 30B+ modellen direct 24GB VRAM verbruiken tijdens het prefillen van lange teksten.
2. **Compatibiliteitsproblemen met community-forks**: Bestaande TurboQuant-branches van de community worden langzaam bijgewerkt en zijn niet compatibel met de nieuwste SWA (Sliding Window) / ISWA hybride cache-architectuur geïntroduceerd in Gemma 4, wat leidt tot `GGML_ASSERT` of segmentatiefouten.
3. **Agent Scheduling Death Loop**: In combinatie met AI-IDE's zoals Claude Code, ZeroClaw of Google Antigravity, veroorzaakt de prefill-latentie van ultralange prompts gemakkelijk client-timeout-retries, waardoor de server in een stroomvretende stationaire doodslus van paniekachtige `cancel task`-acties terechtkomt.

**Dit project biedt een gezuiverde limiet-patch (`llama_turboquant.patch`).** Het injecteert de kern TurboQuant-rotatiecompressie-operators nauwkeurig in de nieuwste officiële mainline-code, waarbij C++-bronconflicten en geheugenuitlijningsfouten handmatig worden opgelost, waardoor crash-redlines perfect worden omzeild.

---

## 📊 Extreme Performance Benchmarks

- **Hardware**: Enkele NVIDIA GeForce RTX 3090 (24GB VRAM)
- **Modellen**: `Gemma-4-31B-It-UD-IQ3_XXS.gguf` / `Qwopus3.5-27B-v3-Q4_K_M.gguf`
- **VRAM-optimalisatie**: 
  - 98.304 (96K) context geïnitialiseerd, VRAM-gebruik vergrendeld op **23.644 MiB / 24.576 MiB**.
  - KV Cache (K+V) gecomprimeerd van >6GB naar **~1.48 GB**.
- **Generatiesnelheid (Decode)**: Volledige `-ngl 99` versnelling, stabiele output op **~16.5 tokens/s**.

---

## 🛠️ Build- & Injectiegids

Om ervoor te zorgen dat de patch perfect wordt toegepast en om compilatiefouten door upstream API-wijzigingen te voorkomen, **dient u uw repository strikt te vergrendelen op de geteste specifieke commit-versie**.

### 1. Repository voorbereiden & versie vergrendelen
```bash
git clone https://github.com/ggml-org/llama.cpp.git
cd llama.cpp

# MOET vergrendeld worden op deze veilige versie (Build b8728)
git checkout 5e9c63546

# Patch toepassen en builden
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
