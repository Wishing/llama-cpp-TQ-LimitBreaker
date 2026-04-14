[ 🇺🇸 English](../README.md) · [ 🇨🇳 简体中文](README_cn.md) · [ 🇯🇵 日本語](README_ja.md) · [ 🇰🇷 한국어](README_ko.md) · [ 🇻🇳 Tiếng Việt](README_vi.md) · [ 🇵🇭 Tagalog](README_tl.md) · [ 🇪🇸 Español](README_es.md) · [ 🇵🇹 Português](README_pt.md) · [ 🇮🇹 Italiano](README_it.md) · [ 🇩🇪 Deutsch](README_de.md) · [ 🇫🇷 Français](README_fr.md) · [ 🇸🇦 العربية](README_ar.md) · [ 🇮🇳 हिन्दी](README_hi.md) · [ 🇷🇺 Русский](README_ru.md) · [ 🇧🇩 বাংলা](README_bn.md) · [ 🇮🇱 עברית](README_he.md) · [ 🇵🇱 Polski](README_pl.md) · [ 🇨🇿 Čeština](README_cs.md) · [ 🇳🇱 Nederlands](README_nl.md) · [ 🇹🇷 Türkçe](README_tr.md) · [ 🇺🇦 Українська](README_uk.md) · [ 🇮🇩 Bahasa Indonesia](README_id.md) · [ 🇹🇭 ไทย](README_th.md) · [ 🇵🇰 اردو](README_ur.md) · [ 🇷🇴 Română](README_ro.md) · [ 🇸🇪 Svenska](README_sv.md) · [ 🇬🇷 Ελληνικά](README_el.md) · [ 🇭🇺 Magyar](README_hu.md) · [ 🇫🇮 Suomi](README_fi.md) · [ 🇩🇰 Dansk](README_da.md) · [ 🇳🇴 Norsk](README_no.md)

# 🚀 llama.cpp TurboQuant Limit Breaker (Έτοιμο για Gemma 4 & Qwopus)

> **Σπάζοντας τους φυσικούς νόμους των 24GB VRAM: Εκτελέστε το Gemma-4-31B / Qwopus-27B με 100% GPU full offload σε μια μοναδική RTX 3090, διατηρώντας παράλληλα μαζικό context 96K - 256K.**

## 🔍 Γιατί αυτό το έργο;

Από τον Απρίλιο του 2026, η τοπική ανάπτυξη της κοινότητας AI ανοιχτού κώδικα αντιμετωπίζει ένα «αδιέξοδο»:
1. **Επίσημη καθυστέρηση**: Το επίσημο mainline του `llama.cpp` δεν έχει ενσωματώσει εγγενώς τους τελεστές TurboQuant (`tbqp3_0`) στο CUDA, προκαλώντας στα μοντέλα 30B+ την άμεση εξάντληση των 24GB VRAM κατά τη διάρκεια του prefill μεγάλου κειμένου.
2. **Προβλήματα συμβατότητας Community Fork**: Τα υπάρχοντα community forks του TurboQuant αργούν να ενημερωθούν και δεν είναι συμβατά με την τελευταία υβριδική αρχιτεκτονική cache SWA (Sliding Window) / ISWA που εισήχθη στο Gemma 4, οδηγώντας σε `GGML_ASSERT` ή σφάλματα τμηματοποίησης (segmentation faults).
3. **Agent Scheduling Death Loop**: Όταν συνδυάζεται με AI IDEs όπως το Claude Code, το ZeroClaw ή το Google Antigravity, η καθυστέρηση prefill πολύ μεγάλων prompts προκαλεί εύκολα επαναλήψεις λόγω timeout του client, με αποτέλεσμα ο διακομιστής να πέφτει σε έναν βρόχο θανάτου υψηλής κατανάλωσης σε αδράνεια με μανιώδεις ακυρώσεις εργασιών (`cancel task`).

**Αυτό το έργο παρέχει ένα καθαρό patch ορίων (`llama_turboquant.patch`).** Εγχέει με ακρίβεια τους βασικούς τελεστές συμπίεσης περιστροφής TurboQuant στον τελευταίο επίσημο κώδικα mainline, επιλύοντας χειροκίνητα τις συγκρούσεις πηγαίου κώδικα C++ και τα σφάλματα ευθυγράμμισης μνήμης, παρακάμπτοντας τέλεια τις κόκκινες γραμμές των κρασαρισμάτων.

---

## 📊 Extreme Performance Benchmarks

- **Υλικό**: Μία NVIDIA GeForce RTX 3090 (24GB VRAM)
- **Μοντέλα**: `Gemma-4-31B-It-UD-IQ3_XXS.gguf` / `Qwopus3.5-27B-v3-Q4_K_M.gguf`
- **Βελτιστοποίηση VRAM**: 
  - Αρχικοποίηση context 98.304 (96K), η χρήση VRAM κλειδώθηκε στα **23.644 MiB / 24.576 MiB**.
  - Η KV Cache (K+V) συμπιέστηκε από >6GB σε **~1.48 GB**.
- **Ταχύτητα παραγωγής (Decode)**: Πλήρης επιτάχυνση `-ngl 99`, σταθερή έξοδος στα **~16.5 tokens/s**.

---

## 🛠️ Οδηγός Build & Injection

Για να διασφαλίσετε ότι το patch εφαρμόζεται τέλεια και για να αποφύγετε σφάλματα μεταγλώττισης από αλλαγές στο upstream API, **παρακαλούμε κλειδώστε αυστηρά το αποθετήριό σας στη δοκιμασμένη συγκεκριμένη έκδοση commit**.

### 1. Προετοιμασία αποθετηρίου & Κλείδωμα έκδοσης
```bash
git clone https://github.com/ggml-org/llama.cpp.git
cd llama.cpp

# ΠΡΕΠΕΙ να κλειδώσει σε αυτήν την ασφαλή έκδοση (Build b8728)
git checkout 5e9c63546

# Εφαρμογή patch και build
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
