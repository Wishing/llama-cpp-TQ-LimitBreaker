[ 🇺🇸 English](README.md) · [ 🇨🇳 简体中文](README_cn.md) · [ 🇯🇵 日本語](README_ja.md) · [ 🇰🇷 한국어](README_ko.md) · [ 🇻🇳 Tiếng Việt](README_vi.md) · [ 🇵🇭 Tagalog](README_tl.md) · [ 🇪🇸 Español](README_es.md) · [ 🇵🇹 Português](README_pt.md) · [ 🇮🇹 Italiano](README_it.md) · [ 🇩🇪 Deutsch](README_de.md) · [ 🇫🇷 Français](README_fr.md) · [ 🇸🇦 العربية](README_ar.md) · [ 🇮🇳 हिन्दी](README_hi.md) · [ 🇷🇺 Русский](README_ru.md) · [ 🇧🇩 বাংলা](README_bn.md) · [ 🇮🇱 עברית](README_he.md) · [ 🇵🇱 Polski](README_pl.md) · [ 🇨🇿 Čeština](README_cs.md) · [ 🇳🇱 Nederlands](README_nl.md) · [ 🇹🇷 Türkçe](README_tr.md) · [ 🇺🇦 Українська](README_uk.md) · [ 🇮🇩 Bahasa Indonesia](README_id.md) · [ 🇹🇭 ไทย](README_th.md) · [ 🇵🇰 اردو](README_ur.md) · [ 🇷🇴 Română](README_ro.md) · [ 🇸🇪 Svenska](README_sv.md) · [ 🇬🇷 Ελληνικά](README_el.md) · [ 🇭🇺 Magyar](README_hu.md) · [ 🇫🇮 Suomi](README_fi.md) · [ 🇩🇰 Dansk](README_da.md) · [ 🇳🇴 Norsk](README_no.md)

# 🚀 llama.cpp TurboQuant Limit Breaker (Gemma 4 اور Qwopus کے لیے تیار)

> **24GB VRAM کے طبعی قوانین کو توڑنا: ایک ہی RTX 3090 پر 100% GPU فل آف لوڈ کے ساتھ Gemma-4-31B / Qwopus-27B چلائیں، جبکہ 96K - 256K بڑے سیاق و سباق (context) کو برقرار رکھیں۔**

## 🔍 یہ پروجیکٹ کیوں؟

اپریل 2026 تک، اوپن سورس AI کمیونٹی کی مقامی تعیناتی کو ایک "ڈیڈ لاک" کا سامنا ہے:
1. **آفیشل تاخیر**: آفیشل `llama.cpp` مین لائن نے TurboQuant آپریٹرز (`tbqp3_0`) کو CUDA میں مقامی طور پر ضم نہیں کیا ہے، جس کی وجہ سے 30B+ ماڈلز طویل متن کے پری فل (prefill) کے دوران فوری طور پر 24GB VRAM ختم کر دیتے ہیں۔
2. **کمیونٹی فورک کی مطابقت کے مسائل**: کمیونٹی کی موجودہ TurboQuant شاخیں اپ ڈیٹ ہونے میں سست ہیں اور Gemma 4 میں متعارف کرائے گئے تازہ ترین SWA (سلائڈنگ ونڈو) / ISWA ہائبرڈ کیشے آرکیٹیکچر کے ساتھ مطابقت نہیں رکھتی ہیں، جس کی وجہ سے `GGML_ASSERT` یا سیگمنٹیشن فالٹس (segmentation faults) ہوتے ہیں۔
3. **ایجنٹ شیڈولنگ ڈیتھ لوپ**: جب Claude Code، ZeroClaw، یا Google Antigravity جیسے AI IDEs کے ساتھ استعمال کیا جاتا ہے، تو انتہائی طویل پرامپٹس کی پری فل لیٹنسی آسانی سے کلائنٹ ٹائم آؤٹ ری ٹرائز کو متحرک کرتی ہے، جس کی وجہ سے سرور جنونی طور پر `cancel task` کے ہائی پاور آئیڈل ڈیتھ لوپ میں گر جاتا ہے۔

**یہ پروجیکٹ ایک پیوریفائیڈ لمیٹ پیچ (`llama_turboquant.patch`) فراہم کرتا ہے۔** یہ تازہ ترین آفیشل مین لائن کوڈ میں بنیادی TurboQuant روٹیشن کمپریشن آپریٹرز کو درست طریقے سے داخل کرتا ہے، C++ سورس تنازعات اور میموری الائنمنٹ کی غلطیوں کو دستی طور پر حل کرتا ہے، اور کریش ریڈ لائنز کو مکمل طور پر بائی پاس کرتا ہے۔

---

## 📊 انتہائی کارکردگی کے بینچ مارکس

- **ہارڈ ویئر**: ایک NVIDIA GeForce RTX 3090 (24GB VRAM)
- **ماڈلز**: `Gemma-4-31B-It-UD-IQ3_XXS.gguf` / `Qwopus3.5-27B-v3-Q4_K_M.gguf`
- **VRAM آپٹیمائزیشن**: 
  - 98,304 (96K) سیاق و سباق شروع کیا گیا، VRAM کا استعمال **23,644 MiB / 24,576 MiB** پر لاک کر دیا گیا۔
  - KV کیشے (K+V) کو >6GB سے **~1.48 GB** تک کمپریس کیا گیا۔
- **پیداواری رفتار (Decode)**: مکمل `-ngl 99` ایکسلریشن، **~16.5 tokens/s** پر مستحکم آؤٹ پٹ۔

---

## 🛠️ بلڈ اور انجکشن گائیڈ

اس بات کو یقینی بنانے کے لیے کہ پیچ بالکل درست طریقے سے لگایا گیا ہے اور اپ اسٹریم API تبدیلیوں سے کمپائلیشن کی غلطیوں سے بچنے کے لیے، **براہ کرم اپنے ریپوزٹری کو ٹیسٹ شدہ مخصوص کمٹ ورژن پر سختی سے لاک کریں**۔

### 1. ریپوزٹری تیار کریں اور ورژن لاک کریں
```bash
git clone https://github.com/ggml-org/llama.cpp.git
cd llama.cpp

# اس محفوظ ورژن پر لاک ہونا چاہیے (Build b8728)
git checkout 5e9c63546

# پیچ لگائیں اور بلڈ کریں
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
