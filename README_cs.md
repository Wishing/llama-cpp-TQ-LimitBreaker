[ 🇺🇸 English](README.md) · [ 🇨🇳 简体中文](README_cn.md) · [ 🇯🇵 日本語](README_ja.md) · [ 🇰🇷 한국어](README_ko.md) · [ 🇻🇳 Tiếng Việt](README_vi.md) · [ 🇵🇭 Tagalog](README_tl.md) · [ 🇪🇸 Español](README_es.md) · [ 🇵🇹 Português](README_pt.md) · [ 🇮🇹 Italiano](README_it.md) · [ 🇩🇪 Deutsch](README_de.md) · [ 🇫🇷 Français](README_fr.md) · [ 🇸🇦 العربية](README_ar.md) · [ 🇮🇳 हिन्दी](README_hi.md) · [ 🇷🇺 Русский](README_ru.md) · [ 🇧🇩 বাংলা](README_bn.md) · [ 🇮🇱 עברית](README_he.md) · [ 🇵🇱 Polski](README_pl.md) · [ 🇨🇿 Čeština](README_cs.md) · [ 🇳🇱 Nederlands](README_nl.md) · [ 🇹🇷 Türkçe](README_tr.md) · [ 🇺🇦 Українська](README_uk.md) · [ 🇮🇩 Bahasa Indonesia](README_id.md) · [ 🇹🇭 ไทย](README_th.md) · [ 🇵🇰 اردو](README_ur.md) · [ 🇷🇴 Română](README_ro.md) · [ 🇸🇪 Svenska](README_sv.md) · [ 🇬🇷 Ελληνικά](README_el.md) · [ 🇭🇺 Magyar](README_hu.md) · [ 🇫🇮 Suomi](README_fi.md) · [ 🇩🇰 Dansk](README_da.md) · [ 🇳🇴 Norsk](README_no.md)

# 🚀 llama.cpp TurboQuant Limit Breaker (Připraveno pro Gemma 4 & Qwopus)

> **Prolomení fyzikálních zákonů 24GB VRAM: Spusťte Gemma-4-31B / Qwopus-27B se 100% GPU full offload na jediné kartě RTX 3090 při zachování masivního kontextu 96K - 256K.**

## 🔍 Proč tento projekt?

K dubnu 2026 čelí lokální nasazení komunity open-source AI „patové situaci“:
1. **Oficiální zpoždění**: Oficiální hlavní větev (mainline) `llama.cpp` nativně neintegrovala operátory TurboQuant (`tbqp3_0`) do CUDA, což způsobuje, že modely 30B+ při prefillu dlouhých textů okamžitě vyčerpají 24GB VRAM.
2. **Problémy s kompatibilitou komunitních forků**: Stávající komunitní větve TurboQuant se aktualizují pomalu a nejsou kompatibilní s nejnovější architekturou hybridní mezipaměti SWA (Sliding Window) / ISWA představenou v Gemma 4, což vede k chybám `GGML_ASSERT` nebo pádům (segmentation faults).
3. **Mrtvá smyčka plánování agentů**: Při spárování s AI IDE, jako jsou Claude Code, ZeroClaw nebo Google Antigravity, latence prefillu ultra dlouhých promptů snadno vyvolá opakované pokusy klienta kvůli vypršení časového limitu, což způsobí, že server upadne do energeticky náročné mrtvé smyčky zběsilého rušení úloh (`cancel task`).

**Tento projekt poskytuje pročištěný limitní patch (`llama_turboquant.patch`).** Přesně vkládá jádro operátorů rotační komprese TurboQuant do nejnovějšího oficiálního kódu hlavní větve, ručně řeší konflikty zdrojového kódu C++ a chyby zarovnání paměti a dokonale obchází kritické body pádů.

---

## 📊 Extrémní výkonnostní benchmarky

- **Hardware**: Jedna karta NVIDIA GeForce RTX 3090 (24GB VRAM)
- **Modely**: `Gemma-4-31B-It-UD-IQ3_XXS.gguf` / `Qwopus3.5-27B-v3-Q4_K_M.gguf`
- **Optimalizace VRAM**: 
  - Inicializován kontext 98 304 (96K), využití VRAM uzamčeno na **23 644 MiB / 24 576 MiB**.
  - KV Cache (K+V) komprimována z >6GB na **~1.48 GB**.
- **Rychlost generování (Decode)**: Plná akcelerace `-ngl 99`, stabilní výstup při **~16.5 tokens/s**.

---

## 🛠️ Průvodce sestavením a injekcí

Aby bylo zajištěno dokonalé použití patche a předešlo se chybám při kompilaci v důsledku změn upstream API, **striktně uzamkněte své úložiště na testované konkrétní verzi commitu**.

### 1. Příprava úložiště a uzamčení verze
```bash
git clone https://github.com/ggml-org/llama.cpp.git
cd llama.cpp

# MUSÍ být uzamčeno na této bezpečné verzi (Build b8728)
git checkout 5e9c63546

# Použít patch a sestavit
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
