[ 🇺🇸 English](../README.md) · [ 🇨🇳 简体中文](README_cn.md) · [ 🇯🇵 日本語](README_ja.md) · [ 🇰🇷 한국어](README_ko.md) · [ 🇻🇳 Tiếng Việt](README_vi.md) · [ 🇵🇭 Tagalog](README_tl.md) · [ 🇪🇸 Español](README_es.md) · [ 🇵🇹 Português](README_pt.md) · [ 🇮🇹 Italiano](README_it.md) · [ 🇩🇪 Deutsch](README_de.md) · [ 🇫🇷 Français](README_fr.md) · [ 🇸🇦 العربية](README_ar.md) · [ 🇮🇳 हिन्दी](README_hi.md) · [ 🇷🇺 Русский](README_ru.md) · [ 🇧🇩 বাংলা](README_bn.md) · [ 🇮🇱 עברית](README_he.md) · [ 🇵🇱 Polski](README_pl.md) · [ 🇨🇿 Čeština](README_cs.md) · [ 🇳🇱 Nederlands](README_nl.md) · [ 🇹🇷 Türkçe](README_tr.md) · [ 🇺🇦 Українська](README_uk.md) · [ 🇮🇩 Bahasa Indonesia](README_id.md) · [ 🇹🇭 ไทย](README_th.md) · [ 🇵🇰 اردو](README_ur.md) · [ 🇷🇴 Română](README_ro.md) · [ 🇸🇪 Svenska](README_sv.md) · [ 🇬🇷 Ελληνικά](README_el.md) · [ 🇭🇺 Magyar](README_hu.md) · [ 🇫🇮 Suomi](README_fi.md) · [ 🇩🇰 Dansk](README_da.md) · [ 🇳🇴 Norsk](README_no.md)

# 🚀 llama.cpp TurboQuant Limit Breaker (מוכן ל-Gemma 4 ו-Qwopus)

> **שבירת חוקי הפיזיקה של 24GB VRAM: הרץ את Gemma-4-31B / Qwopus-27B עם 100% GPU full offload על כרטיס RTX 3090 יחיד, תוך שמירה על קונטקסט עצום של 96K - 256K.**

## 🔍 למה הפרויקט הזה?

נכון לאפריל 2026, הפריסה המקומית של קהילת ה-AI בקוד פתוח עומדת בפני "מבוי סתום":
1. **פיגור רשמי**: הערוץ הראשי (mainline) הרשמי של `llama.cpp` לא שילב באופן טבעי את אופרטורי TurboQuant (`tbqp3_0`) ב-CUDA, מה שגורם למודלים של 30B+ לכלות באופן מיידי 24GB VRAM במהלך prefill של טקסטים ארוכים.
2. **בעיות תאימות של פיצולים בקהילה**: פיצולי TurboQuant קיימים בקהילה איטיים לעדכון ואינם תואמים לארכיטקטורת ה-cache ההיברידית SWA (Sliding Window) / ISWA העדכנית ביותר שהוצגה ב-Gemma 4, מה שמוביל לשגיאות `GGML_ASSERT` או קריסות (segmentation faults).
3. **לולאת מוות בתזמון סוכנים**: כשמשלבים את המודלים עם סביבות פיתוח AI (IDEs) כמו Claude Code, ZeroClaw או Google Antigravity, השיהוי ב-prefill של פרומפטים ארוכים מאוד גורם בקלות לניסיונות חוזרים של הלקוח עקב timeout, מה שגורם לשרת ליפול ללולאת מוות של צריכת חשמל גבוהה במצב סרק עם ביטולי משימות (`cancel task`) תזזיתיים.

**פרויקט זה מספק טלאי (patch) מגבלות מזוקק (`llama_turboquant.patch`).** הוא מזריק במדויק את אופרטורי דחיסת הסיבוב של TurboQuant לקוד ה-mainline הרשמי העדכני ביותר, תוך פתרון ידני של קונפליקטים בקוד המקור ב-C++ ושגיאות יישור זיכרון, ועוקף בצורה מושלמת את הקווים האדומים של הקריסות.

---

## 📊 מדדי ביצועים קיצוניים

- **חומרה**: כרטיס NVIDIA GeForce RTX 3090 יחיד (24GB VRAM)
- **מודלים**: `Gemma-4-31B-It-UD-IQ3_XXS.gguf` / `Qwopus3.5-27B-v3-Q4_K_M.gguf`
- **אופטימיזציית VRAM**: 
  - הופעל קונטקסט של 98,304 (96K), שימוש ב-VRAM ננעל על **23,644 MiB / 24,576 MiB**.
  - KV Cache (K+V) נדחס מ- >6GB ל- **~1.48 GB**.
- **מהירות יצירה (Decode)**: האצה מלאה של `-ngl 99`, פלט יציב של **~16.5 tokens/s**.

---

## 🛠️ מדריך בנייה והזרקה

כדי להבטיח שהטלאי יוחל בצורה מושלמת וכדי למנוע שגיאות הידור עקב שינויים ב-API של המקור, **אנא נעל את המאגר שלך לגרסת ה-commit הספציפית שנבדקה**.

### 1. הכנת המאגר ונעילת גרסה
```bash
git clone https://github.com/ggml-org/llama.cpp.git
cd llama.cpp

# חייב להינעל לגרסה בטוחה זו (Build b8728)
git checkout 5e9c63546

# החלת הטלאי ובנייה
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
