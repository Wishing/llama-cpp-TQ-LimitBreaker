[ 🇺🇸 English](README.md) · [ 🇨🇳 简体中文](README_cn.md) · [ 🇯🇵 日本語](README_ja.md) · [ 🇰🇷 한국어](README_ko.md) · [ 🇻🇳 Tiếng Việt](README_vi.md) · [ 🇵🇭 Tagalog](README_tl.md) · [ 🇪🇸 Español](README_es.md) · [ 🇵🇹 Português](README_pt.md) · [ 🇮🇹 Italiano](README_it.md) · [ 🇩🇪 Deutsch](README_de.md) · [ 🇫🇷 Français](README_fr.md) · [ 🇸🇦 العربية](README_ar.md) · [ 🇮🇳 हिन्दी](README_hi.md) · [ 🇷🇺 Русский](README_ru.md) · [ 🇧🇩 বাংলা](README_bn.md) · [ 🇮🇱 עברית](README_he.md) · [ 🇵🇱 Polski](README_pl.md) · [ 🇨🇿 Čeština](README_cs.md) · [ 🇳🇱 Nederlands](README_nl.md) · [ 🇹🇷 Türkçe](README_tr.md) · [ 🇺🇦 Українська](README_uk.md) · [ 🇮🇩 Bahasa Indonesia](README_id.md) · [ 🇹🇭 ไทย](README_th.md) · [ 🇵🇰 اردو](README_ur.md) · [ 🇷🇴 Română](README_ro.md) · [ 🇸🇪 Svenska](README_sv.md) · [ 🇬🇷 Ελληνικά](README_el.md) · [ 🇭🇺 Magyar](README_hu.md) · [ 🇫🇮 Suomi](README_fi.md) · [ 🇩🇰 Dansk](README_da.md) · [ 🇳🇴 Norsk](README_no.md)

# 🚀 llama.cpp TurboQuant Limit Breaker (Gemma 4 & Qwopus valmiina)

> **24 Gt VRAM:n fysiikan lakien rikkominen: Aja Gemma-4-31B / Qwopus-27B 100 % GPU full offloadilla yhdellä RTX 3090:llä säilyttäen samalla massiivisen 96K - 256K kontekstin.**

## 🔍 Miksi tämä projekti?

Huhtikuusta 2026 lähtien avoimen lähdekoodin AI-yhteisön paikallinen käyttöönotto on kohdannut "umpikujan":
1. **Virallinen viive**: Virallinen `llama.cpp`-päähaara ei ole integroinut TurboQuant-operaattoreita (`tbqp3_0`) CUDAan natiivisti, mikä saa 30B+ -mallit kuluttamaan välittömästi 24 Gt VRAMia pitkän tekstin esitäytön (prefill) aikana.
2. **Yhteisön haarautumien (fork) yhteensopivuusongelmat**: Nykyiset yhteisön TurboQuant-haarat päivittyvät hitaasti eivätkä ole yhteensopivia Gemma 4:ssä esitellyn uusimman SWA (Sliding Window) / ISWA -hybridivälimuistiarkkitehtuurin kanssa, mikä johtaa `GGML_ASSERT`-virheisiin tai muistivirheisiin (segmentation faults).
3. **Agent Scheduling Death Loop**: Kun käytetään AI-IDE-työkaluja, kuten Claude Code, ZeroClaw tai Google Antigravity, erittäin pitkien promptien esitäytön viive laukaisee helposti asiakkaan aikakatkaisun uudelleenyritykset, mikä saa palvelimen joutumaan korkean tehon joutokäynnin kuoleman kierteeseen ja peruuttamaan tehtäviä (`cancel task`) jatkuvasti.

**Tämä projekti tarjoaa puhdistetun rajoituspaikkauksen (`llama_turboquant.patch`).** Se injektoi tarkasti TurboQuant-kiertopakkausoperaattorit uusimpaan viralliseen pääkoodiin, ratkaisee manuaalisesti C++-lähdekoodikonfliktit ja muistin kohdistusvirheet, ohittaen täydellisesti kaatumisrajat.

---

## 📊 Äärimmäiset suorituskykytestit

- **Laitteisto**: Yksi NVIDIA GeForce RTX 3090 (24 Gt VRAM)
- **Mallit**: `Gemma-4-31B-It-UD-IQ3_XXS.gguf` / `Qwopus3.5-27B-v3-Q4_K_M.gguf`
- **VRAM-optimointi**: 
  - 98 304 (96K) konteksti alustettu, VRAM-käyttö lukittu arvoon **23 644 MiB / 24 576 MiB**.
  - KV Cache (K+V) pakattu yli 6 Gt:sta **noin 1,48 Gt:aan**.
- **Sukupolven nopeus (Decode)**: Täysi `-ngl 99` -kiihdytys, vakaa ulostulo nopeudella **n. 16,5 tokens/s**.

---

## 🛠️ Rakennus- ja injektio-opas

Jotta paikkaus voidaan soveltaa täydellisesti ja välttää ylävirran API-muutoksista johtuvat käännösvirheet, **lukitse arkistosi tiukasti testattuun tiettyyn commit-versioon**.

### 1. Valmistele arkisto ja lukitse versio
```bash
git clone https://github.com/ggml-org/llama.cpp.git
cd llama.cpp

# TÄYTYY lukita tähän turvalliseen versioon (Build b8728)
git checkout 5e9c63546

# Käytä paikkausta ja rakenna
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
