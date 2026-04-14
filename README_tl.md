[ 🇺🇸 English](README.md) · [ 🇨🇳 简体中文](README_cn.md) · [ 🇯🇵 日本語](README_ja.md) · [ 🇰🇷 한국어](README_ko.md) · [ 🇻🇳 Tiếng Việt](README_vi.md) · [ 🇵🇭 Tagalog](README_tl.md) · [ 🇪🇸 Español](README_es.md) · [ 🇵🇹 Português](README_pt.md) · [ 🇮🇹 Italiano](README_it.md) · [ 🇩🇪 Deutsch](README_de.md) · [ 🇫🇷 Français](README_fr.md) · [ 🇸🇦 العربية](README_ar.md) · [ 🇮🇳 हिन्दी](README_hi.md) · [ 🇷🇺 Русский](README_ru.md) · [ 🇧🇩 বাংলা](README_bn.md) · [ 🇮🇱 עברית](README_he.md) · [ 🇵🇱 Polski](README_pl.md) · [ 🇨🇿 Čeština](README_cs.md) · [ 🇳🇱 Nederlands](README_nl.md) · [ 🇹🇷 Türkçe](README_tr.md) · [ 🇺🇦 Українська](README_uk.md) · [ 🇮🇩 Bahasa Indonesia](README_id.md) · [ 🇹🇭 ไทย](README_th.md) · [ 🇵🇰 اردو](README_ur.md) · [ 🇷🇴 Română](README_ro.md) · [ 🇸🇪 Svenska](README_sv.md) · [ 🇬🇷 Ελληνικά](README_el.md) · [ 🇭🇺 Magyar](README_hu.md) · [ 🇫🇮 Suomi](README_fi.md) · [ 🇩🇰 Dansk](README_da.md) · [ 🇳🇴 Norsk](README_no.md)

# 🚀 llama.cpp TurboQuant Limit Breaker (Handa na para sa Gemma 4 at Qwopus)

> **Pagbasag sa mga Batas ng Pisika ng 24GB VRAM: Patakbuhin ang Gemma-4-31B / Qwopus-27B na may 100% GPU full offload sa isang RTX 3090, habang pinapanatili ang 96K - 256K na malawak na context.**

## 🔍 Bakit ang proyektong ito?

Mula noong Abril 2026, ang lokal na deployment ng open-source AI community ay nahaharap sa isang "deadlock":
1. **Opisyal na Pagkaantala**: Ang opisyal na `llama.cpp` mainline ay hindi pa natively integrated ang TurboQuant operators (`tbqp3_0`) sa CUDA, na nagiging sanhi ng mabilis na pagkaubos ng 24GB VRAM sa mga 30B+ models habang nagpe-prefill ng mahabang text.
2. **Mga Isyu sa Kompatibilidad ng Community Fork**: Ang mga umiiral na TurboQuant branches sa komunidad ay mabagal mag-update at hindi kompatibol sa pinakabagong SWA (Sliding Window) / ISWA hybrid cache architecture na ipinakilala sa Gemma 4, na nagreresulta sa `GGML_ASSERT` o segmentation faults.
3. **Agent Scheduling Death Loop**: Kapag ipinares sa mga AI IDE tulad ng Claude Code, ZeroClaw, o Google Antigravity, ang prefill latency ng napakahahabang prompts ay madaling nagti-trigger ng mga client timeout retries, na nagiging sanhi ng pagkakakulong ng server sa isang high-power idle death loop ng paulit-ulit na `cancel task`.

**Ang proyektong ito ay nagbibigay ng isang purified limit patch (`llama_turboquant.patch`).** Tumpak nitong ipinapasok ang core TurboQuant rotation compression operators sa pinakabagong opisyal na mainline code, habang manu-manong inaayos ang mga C++ source conflicts at memory alignment errors, upang perpektong malampasan ang mga crash red lines.

---

## 📊 Extreme Performance Benchmarks

- **Hardware**: Isang NVIDIA GeForce RTX 3090 (24GB VRAM)
- **Mga Modelo**: `Gemma-4-31B-It-UD-IQ3_XXS.gguf` / `Qwopus3.5-27B-v3-Q4_K_M.gguf`
- **VRAM Optimization**: 
  - 98,304 (96K) context initialized, ang paggamit ng VRAM ay naka-lock sa **23,644 MiB / 24,576 MiB**.
  - KV Cache (K+V) compressed mula >6GB patungong **~1.48 GB**.
- **Bilis ng Generasyon (Decode)**: Full `-ngl 99` acceleration, stable na output sa **~16.5 tokens/s**.

---

## 🛠️ Gabay sa Build at Injection

Upang matiyak na ang patch ay nailapat nang perpekto at maiwasan ang mga errors sa compilation mula sa mga pagbabago sa upstream API, **mangyaring mahigpit na i-lock ang iyong repository sa sinubukang partikular na commit version**.

### 1. Ihanda ang Repository at I-lock ang Bersyon
```bash
git clone https://github.com/ggml-org/llama.cpp.git
cd llama.cpp

# DAPAT i-lock sa ligtas na bersyong ito (Build b8728)
git checkout 5e9c63546

# Ilapat ang patch at i-build
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
