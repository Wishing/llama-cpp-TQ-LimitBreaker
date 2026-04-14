[ 🇺🇸 English](README.md) · [ 🇨🇳 简体中文](README_cn.md) · [ 🇯🇵 日本語](README_ja.md) · [ 🇰🇷 한국어](README_ko.md) · [ 🇻🇳 Tiếng Việt](README_vi.md) · [ 🇵🇭 Tagalog](README_tl.md) · [ 🇪🇸 Español](README_es.md) · [ 🇵🇹 Português](README_pt.md) · [ 🇮🇹 Italiano](README_it.md) · [ 🇩🇪 Deutsch](README_de.md) · [ 🇫🇷 Français](README_fr.md) · [ 🇸🇦 العربية](README_ar.md) · [ 🇮🇳 हिन्दी](README_hi.md) · [ 🇷🇺 Русский](README_ru.md) · [ 🇧🇩 বাংলা](README_bn.md) · [ 🇮🇱 עברית](README_he.md) · [ 🇵🇱 Polski](README_pl.md) · [ 🇨🇿 Čeština](README_cs.md) · [ 🇳🇱 Nederlands](README_nl.md) · [ 🇹🇷 Türkçe](README_tr.md) · [ 🇺🇦 Українська](README_uk.md) · [ 🇮🇩 Bahasa Indonesia](README_id.md) · [ 🇹🇭 ไทย](README_th.md) · [ 🇵🇰 اردو](README_ur.md) · [ 🇷🇴 Română](README_ro.md) · [ 🇸🇪 Svenska](README_sv.md) · [ 🇬🇷 Ελληνικά](README_el.md) · [ 🇭🇺 Magyar](README_hu.md) · [ 🇫🇮 Suomi](README_fi.md) · [ 🇩🇰 Dansk](README_da.md) · [ 🇳🇴 Norsk](README_no.md)

# 🚀 llama.cpp TurboQuant Limit Breaker (Gotowy na Gemma 4 i Qwopus)

> **Przełamywanie praw fizyki 24GB VRAM: Uruchom Gemma-4-31B / Qwopus-27B ze 100% odciążeniem GPU na pojedynczej karcie RTX 3090, zachowując ogromny kontekst 96K - 256K.**

## 🔍 Dlaczego ten projekt?

Od kwietnia 2026 r. lokalne wdrażanie społeczności AI o otwartym kodzie źródłowym stoi w obliczu „impasu”:
1. **Oficjalne opóźnienie**: Oficjalna linia główna `llama.cpp` nie zintegrowała natywnie operatorów TurboQuant (`tbqp3_0`) z CUDA, co powoduje, że modele 30B+ błyskawicznie zużywają 24 GB pamięci VRAM podczas wstępnego wypełniania (prefill) długich tekstów.
2. **Problemy z kompatybilnością forków społeczności**: Istniejące gałęzie TurboQuant społeczności są aktualizowane powoli i nie są kompatybilne z najnowszą hybrydową architekturą pamięci podręcznej SWA (Sliding Window) / ISWA wprowadzoną w Gemma 4, co prowadzi do błędów `GGML_ASSERT` lub błędów segmentacji.
3. **Pętla śmierci harmonogramowania agentów**: W połączeniu z AI IDE, takimi jak Claude Code, ZeroClaw lub Google Antigravity, opóźnienie prefill ultra-długich promptów łatwo wyzwala ponowne próby klienta z powodu przekroczenia limitu czasu, co powoduje, że serwer wpada w pętlę śmierci wysokiego zużycia energii w stanie bezczynności z gorączkowymi próbami `cancel task`.

**Ten projekt zapewnia oczyszczony patch limitu (`llama_turboquant.patch`).** Precyzyjnie wstrzykuje on podstawowe operatory kompresji rotacyjnej TurboQuant do najnowszego oficjalnego kodu linii głównej, ręcznie rozwiązując konflikty źródeł C++ i błędy wyrównania pamięci, idealnie omijając czerwone linie awarii.

---

## 📊 Ekstremalne testy wydajnościowe

- **Sprzęt**: Pojedyncza karta NVIDIA GeForce RTX 3090 (24 GB VRAM)
- **Modele**: `Gemma-4-31B-It-UD-IQ3_XXS.gguf` / `Qwopus3.5-27B-v3-Q4_K_M.gguf`
- **Optymalizacja VRAM**: 
  - Zainicjowano kontekst 98 304 (96K), użycie pamięci VRAM zablokowane na poziomie **23 644 MiB / 24 576 MiB**.
  - Pamięć podręczna KV Cache (K+V) skompresowana z >6 GB do **~1,48 GB**.
- **Prędkość generowania (Decode)**: Pełne przyspieszenie `-ngl 99`, stabilna moc wyjściowa na poziomie **~16,5 tokena/s**.

---

## 🛠️ Przewodnik budowania i iniekcji

Aby upewnić się, że poprawka została zastosowana idealnie i uniknąć błędów kompilacji wynikających ze zmian w API upstream, **należy ściśle zablokować repozytorium na przetestowanej, konkretnej wersji commitu**.

### 1. Przygotuj repozytorium i zablokuj wersję
```bash
git clone https://github.com/ggml-org/llama.cpp.git
cd llama.cpp

# MUSI być zablokowany na tej bezpiecznej wersji (Build b8728)
git checkout 5e9c63546

# Zastosuj patch i zbuduj
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
