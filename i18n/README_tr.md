[ 🇺🇸 English](../README.md) · [ 🇨🇳 简体中文](README_cn.md) · [ 🇯🇵 日本語](README_ja.md) · [ 🇰🇷 한국어](README_ko.md) · [ 🇻🇳 Tiếng Việt](README_vi.md) · [ 🇵🇭 Tagalog](README_tl.md) · [ 🇪🇸 Español](README_es.md) · [ 🇵🇹 Português](README_pt.md) · [ 🇮🇹 Italiano](README_it.md) · [ 🇩🇪 Deutsch](README_de.md) · [ 🇫🇷 Français](README_fr.md) · [ 🇸🇦 العربية](README_ar.md) · [ 🇮🇳 हिन्दी](README_hi.md) · [ 🇷🇺 Русский](README_ru.md) · [ 🇧🇩 বাংলা](README_bn.md) · [ 🇮🇱 עברית](README_he.md) · [ 🇵🇱 Polski](README_pl.md) · [ 🇨🇿 Čeština](README_cs.md) · [ 🇳🇱 Nederlands](README_nl.md) · [ 🇹🇷 Türkçe](README_tr.md) · [ 🇺🇦 Українська](README_uk.md) · [ 🇮🇩 Bahasa Indonesia](README_id.md) · [ 🇹🇭 ไทย](README_th.md) · [ 🇵🇰 اردو](README_ur.md) · [ 🇷🇴 Română](README_ro.md) · [ 🇸🇪 Svenska](README_sv.md) · [ 🇬🇷 Ελληνικά](README_el.md) · [ 🇭🇺 Magyar](README_hu.md) · [ 🇫🇮 Suomi](README_fi.md) · [ 🇩🇰 Dansk](README_da.md) · [ 🇳🇴 Norsk](README_no.md)

# 🚀 llama.cpp TurboQuant Limit Breaker (Gemma 4 ve Qwopus Hazır)

> **24GB VRAM'in Fizik Kurallarını Bozmak: Tek bir RTX 3090 üzerinde %100 GPU tam boşaltma (full offload) ile Gemma-4-31B / Qwopus-27B çalıştırın ve 96K - 256K devasa bağlamı koruyun.**

## 🔍 Neden bu proje?

Nisan 2026 itibarıyla, açık kaynak AI topluluğunun yerel dağıtımı bir "çıkmaz" ile karşı karşıya:
1. **Resmi Gecikme**: Resmi `llama.cpp` ana hattı TurboQuant operatörlerini (`tbqp3_0`) CUDA'ya yerel olarak entegre etmedi, bu da 30B+ modellerin uzun metin ön dolumu (prefill) sırasında 24GB VRAM'i anında tüketmesine neden oluyor.
2. **Topluluk Fork Uyumluluk Sorunları**: Mevcut topluluk TurboQuant dalları yavaş güncelleniyor ve Gemma 4'te tanıtılan en son SWA (Sliding Window) / ISWA hibrit önbellek mimarisiyle uyumlu değil; bu da `GGML_ASSERT` veya segmentasyon hatalarına yol açıyor.
3. **Aracı Zamanlama Ölüm Döngüsü**: Claude Code, ZeroClaw veya Google Antigravity gibi AI IDE'lerle eşleştirildiğinde, ultra uzun istemlerin ön dolum gecikmesi istemci zaman aşımı yeniden denemelerini kolayca tetikler ve sunucunun çılgınca `cancel task` yaptığı yüksek güçlü bir boşta ölüm döngüsüne girmesine neden olur.

**Bu proje, arındırılmış bir limit yaması (`llama_turboquant.patch`) sağlar.** Çekirdek TurboQuant rotasyon sıkıştırma operatörlerini en son resmi ana hat koduna tam olarak enjekte eder, C++ kaynak çakışmalarını ve bellek hizalama hatalarını manuel olarak çözer ve çökme kırmızı hatlarını mükemmel bir şekilde baypas eder.

---

## 📊 Ekstrem Performans Kıyaslamaları

- **Donanım**: Tek NVIDIA GeForce RTX 3090 (24GB VRAM)
- **Modeller**: `Gemma-4-31B-It-UD-IQ3_XXS.gguf` / `Qwopus3.5-27B-v3-Q4_K_M.gguf`
- **VRAM Optimizasyonu**: 
  - 98.304 (96K) bağlam başlatıldı, VRAM kullanımı **23.644 MiB / 24.576 MiB** seviyesinde kilitlendi.
  - KV Cache (K+V), >6GB'dan **~1.48 GB**'a sıkıştırıldı.
- **Üretim Hızı (Decode)**: Tam `-ngl 99` hızlandırma, **~16.5 tokens/s**'de kararlı çıktı.

---

## 🛠️ Kurulum ve Enjeksiyon Kılavuzu

Yamanın mükemmel bir şekilde uygulandığından emin olmak ve upstream API değişikliklerinden kaynaklanan derleme hatalarını önlemek için, **lütfen deponuzu test edilen belirli commit sürümüne kesin olarak kilitleyin**.

### 1. Depoyu Hazırlama ve Sürümü Kilitleme
```bash
git clone https://github.com/ggml-org/llama.cpp.git
cd llama.cpp

# Bu güvenli sürüme kilitlenmelidir (Build b8728)
git checkout 5e9c63546

# Yamayı uygula ve derle
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
