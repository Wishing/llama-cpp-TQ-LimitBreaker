[ 🇺🇸 English](../README.md) · [ 🇨🇳 简体中文](README_cn.md) · [ 🇯🇵 日本語](README_ja.md) · [ 🇰🇷 한국어](README_ko.md) · [ 🇻🇳 Tiếng Việt](README_vi.md) · [ 🇵🇭 Tagalog](README_tl.md) · [ 🇪🇸 Español](README_es.md) · [ 🇵🇹 Português](README_pt.md) · [ 🇮🇹 Italiano](README_it.md) · [ 🇩🇪 Deutsch](README_de.md) · [ 🇫🇷 Français](README_fr.md) · [ 🇸🇦 العربية](README_ar.md) · [ 🇮🇳 हिन्दी](README_hi.md) · [ 🇷🇺 Русский](README_ru.md) · [ 🇧🇩 বাংলা](README_bn.md) · [ 🇮🇱 עברית](README_he.md) · [ 🇵🇱 Polski](README_pl.md) · [ 🇨🇿 Čeština](README_cs.md) · [ 🇳🇱 Nederlands](README_nl.md) · [ 🇹🇷 Türkçe](README_tr.md) · [ 🇺🇦 Українська](README_uk.md) · [ 🇮🇩 Bahasa Indonesia](README_id.md) · [ 🇹🇭 ไทย](README_th.md) · [ 🇵🇰 اردو](README_ur.md) · [ 🇷🇴 Română](README_ro.md) · [ 🇸🇪 Svenska](README_sv.md) · [ 🇬🇷 Ελληνικά](README_el.md) · [ 🇭🇺 Magyar](README_hu.md) · [ 🇫🇮 Suomi](README_fi.md) · [ 🇩🇰 Dansk](README_da.md) · [ 🇳🇴 Norsk](README_no.md)

# 🚀 llama.cpp TurboQuant Limit Breaker (جاهز لـ Gemma 4 و Qwopus)

> **كسر قوانين الفيزياء لـ 24 جيجابايت من VRAM: قم بتشغيل Gemma-4-31B / Qwopus-27B مع تفريغ كامل لـ GPU بنسبة 100% على بطاقة RTX 3090 واحدة، مع الحفاظ على سياق ضخم يتراوح بين 96K و 256K.**

## 🔍 لماذا هذا المشروع؟

اعتبارًا من أبريل 2026، يواجه النشر المحلي لمجتمع الذكاء الاصطناعي مفتوح المصدر "طريقًا مسدودًا":
1. **التأخر الرسمي**: لم يقم الخط الرئيسي الرسمي لـ `llama.cpp` بدمج مشغلات TurboQuant (`tbqp3_0`) بشكل أصلي في CUDA، مما يتسبب في استهلاك طرازات +30B لـ 24 جيجابايت من VRAM على الفور أثناء التعبئة المسبقة (prefill) للنصوص الطويلة.
2. **مشكلات توافق فروع المجتمع**: فروع TurboQuant الحالية في المجتمع بطيئة في التحديث ولا تتوافق مع أحدث بنية ذاكرة التخزين المؤقت الهجينة SWA (Sliding Window) / ISWA التي تم تقديمها في Gemma 4، مما يؤدي إلى أخطاء `GGML_ASSERT` أو أخطاء في تقسيم الذاكرة (segmentation faults).
3. **حلقة الموت لجدولة الوكيل (Agent)**: عند استخدامه مع بيئات تطوير الذكاء الاصطناعي (AI IDEs) مثل Claude Code أو ZeroClaw أو Google Antigravity، فإن زمن انتقال التعبئة المسبقة للمطالبات الطويلة جدًا يؤدي بسهولة إلى تحفيز محاولات إعادة محاولة مهلة العميل، مما يتسبب في سقوط الخادم في حلقة موت خاملة عالية الطاقة من عمليات `cancel task` المحمومة.

**يوفر هذا المشروع رقعة حدودية منقاة (`llama_turboquant.patch`).** تقوم بحقن مشغلات ضغط دوران TurboQuant الأساسية بدقة في أحدث كود للخط الرئيسي الرسمي، مع حل تعارضات كود المصدر C++ وأخطاء محاذاة الذاكرة يدويًا، وتجاوز الخطوط الحمراء للانهيار تمامًا.

---

## 📊 مقاييس الأداء القصوى

- **الأجهزة**: بطاقة NVIDIA GeForce RTX 3090 واحدة (24 جيجابايت VRAM)
- **النماذج**: `Gemma-4-31B-It-UD-IQ3_XXS.gguf` / `Qwopus3.5-27B-v3-Q4_K_M.gguf`
- **تحسين VRAM**: 
  - تم تهيئة سياق 98,304 (96K)، وتم قفل استخدام VRAM عند **23,644 ميجابايت / 24,576 ميجابايت**.
  - تم ضغط KV Cache (K+V) من أكثر من 6 جيجابايت إلى **حوالي 1.48 جيجابايت**.
- **سرعة التوليد (Decode)**: تسريع كامل `-ngl 99` ، إخراج مستقر عند **~16.5 tokens/s**.

---

## 🛠️ دليل البناء والحقن

لضمان تطبيق الرقعة بشكل مثالي وتجنب أخطاء الترجمة الناتجة عن تغييرات واجهة برمجة تطبيقات المنبع (upstream API)، **يرجى قفل المستودع الخاص بك بدقة على إصدار الالتزام (commit) المحدد الذي تم اختباره**.

### 1. إعداد المستودع وقفل الإصدار
```bash
git clone https://github.com/ggml-org/llama.cpp.git
cd llama.cpp

# يجب القفل على هذا الإصدار الآمن (Build b8728)
git checkout 5e9c63546

# تطبيق الرقعة والبناء
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
