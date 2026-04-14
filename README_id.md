[ 🇺🇸 English](README.md) · [ 🇨🇳 简体中文](README_cn.md) · [ 🇯🇵 日本語](README_ja.md) · [ 🇰🇷 한국어](README_ko.md) · [ 🇻🇳 Tiếng Việt](README_vi.md) · [ 🇵🇭 Tagalog](README_tl.md) · [ 🇪🇸 Español](README_es.md) · [ 🇵🇹 Português](README_pt.md) · [ 🇮🇹 Italiano](README_it.md) · [ 🇩🇪 Deutsch](README_de.md) · [ 🇫🇷 Français](README_fr.md) · [ 🇸🇦 العربية](README_ar.md) · [ 🇮🇳 हिन्दी](README_hi.md) · [ 🇷🇺 Русский](README_ru.md) · [ 🇧🇩 বাংলা](README_bn.md) · [ 🇮🇱 עברית](README_he.md) · [ 🇵🇱 Polski](README_pl.md) · [ 🇨🇿 Čeština](README_cs.md) · [ 🇳🇱 Nederlands](README_nl.md) · [ 🇹🇷 Türkçe](README_tr.md) · [ 🇺🇦 Українська](README_uk.md) · [ 🇮🇩 Bahasa Indonesia](README_id.md) · [ 🇹🇭 ไทย](README_th.md) · [ 🇵🇰 اردو](README_ur.md) · [ 🇷🇴 Română](README_ro.md) · [ 🇸🇪 Svenska](README_sv.md) · [ 🇬🇷 Ελληνικά](README_el.md) · [ 🇭🇺 Magyar](README_hu.md) · [ 🇫🇮 Suomi](README_fi.md) · [ 🇩🇰 Dansk](README_da.md) · [ 🇳🇴 Norsk](README_no.md)

# 🚀 llama.cpp TurboQuant Limit Breaker (Siap untuk Gemma 4 & Qwopus)

> **Mendobrak Hukum Fisika VRAM 24GB: Jalankan Gemma-4-31B / Qwopus-27B dengan 100% GPU full offload pada satu kartu RTX 3090, sambil mempertahankan konteks masif 96K - 256K.**

## 🔍 Mengapa proyek ini?

Hingga April 2026, penerapan lokal komunitas AI sumber terbuka menghadapi "kebuntuan":
1. **Keterlambatan Resmi**: Mainline `llama.cpp` resmi belum mengintegrasikan operator TurboQuant (`tbqp3_0`) ke dalam CUDA secara native, menyebabkan model 30B+ secara instan menghabiskan VRAM 24GB selama prefill teks panjang.
2. **Masalah Kompatibilitas Fork Komunitas**: Cabang TurboQuant komunitas yang ada lambat untuk diperbarui dan tidak kompatibel dengan arsitektur cache hybrid SWA (Sliding Window) / ISWA terbaru yang diperkenalkan di Gemma 4, yang menyebabkan `GGML_ASSERT` atau kegagalan segmentasi (segmentation faults).
3. **Agent Scheduling Death Loop**: Saat dipasangkan dengan AI IDE seperti Claude Code, ZeroClaw, atau Google Antigravity, latensi prefill dari prompt ultra-panjang dengan mudah memicu upaya ulang timeout klien, menyebabkan server jatuh ke dalam death loop idle berdaya tinggi karena `cancel task` yang panik.

**Proyek ini menyediakan patch batas yang dimurnikan (`llama_turboquant.patch`).** Ini secara tepat menyuntikkan operator kompresi rotasi TurboQuant inti ke dalam kode mainline resmi terbaru, secara manual menyelesaikan konflik sumber C++ dan kesalahan penyelarasan memori, dengan sempurna melewati garis merah kegagalan.

---

## 📊 Tolok Ukur Performa Ekstrem

- **Perangkat Keras**: Satu NVIDIA GeForce RTX 3090 (24GB VRAM)
- **Model**: `Gemma-4-31B-It-UD-IQ3_XXS.gguf` / `Qwopus3.5-27B-v3-Q4_K_M.gguf`
- **Optimalisasi VRAM**: 
  - Konteks 98.304 (96K) diinisialisasi, penggunaan VRAM dikunci pada **23.644 MiB / 24.576 MiB**.
  - KV Cache (K+V) dikompresi dari >6GB menjadi **~1,48 GB**.
- **Kecepatan Generasi (Decode)**: Akselerasi `-ngl 99` penuh, output stabil pada **~16,5 tokens/s**.

---

## 🛠️ Panduan Membangun & Injeksi

Untuk memastikan patch diterapkan dengan sempurna dan untuk menghindari kesalahan kompilasi dari perubahan API hulu, **harap kunci repositori Anda secara ketat ke versi commit tertentu yang telah diuji**.

### 1. Siapkan Repositori & Kunci Versi
```bash
git clone https://github.com/ggml-org/llama.cpp.git
cd llama.cpp

# HARUS dikunci ke versi aman ini (Build b8728)
git checkout 5e9c63546

# Terapkan patch dan bangun
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
