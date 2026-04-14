[ 🇺🇸 English](README.md) · [ 🇨🇳 简体中文](README_cn.md) · [ 🇯🇵 日本語](README_ja.md) · [ 🇰🇷 한국어](README_ko.md) · [ 🇻🇳 Tiếng Việt](README_vi.md) · [ 🇵🇭 Tagalog](README_tl.md) · [ 🇪🇸 Español](README_es.md) · [ 🇵🇹 Português](README_pt.md) · [ 🇮🇹 Italiano](README_it.md) · [ 🇩🇪 Deutsch](README_de.md) · [ 🇫🇷 Français](README_fr.md) · [ 🇸🇦 العربية](README_ar.md) · [ 🇮🇳 हिन्दी](README_hi.md) · [ 🇷🇺 Русский](README_ru.md) · [ 🇧🇩 বাংলা](README_bn.md) · [ 🇮🇱 עברית](README_he.md) · [ 🇵🇱 Polski](README_pl.md) · [ 🇨🇿 Čeština](README_cs.md) · [ 🇳🇱 Nederlands](README_nl.md) · [ 🇹🇷 Türkçe](README_tr.md) · [ 🇺🇦 Українська](README_uk.md) · [ 🇮🇩 Bahasa Indonesia](README_id.md) · [ 🇹🇭 ไทย](README_th.md) · [ 🇵🇰 اردو](README_ur.md) · [ 🇷🇴 Română](README_ro.md) · [ 🇸🇪 Svenska](README_sv.md) · [ 🇬🇷 Ελληνικά](README_el.md) · [ 🇭🇺 Magyar](README_hu.md) · [ 🇫🇮 Suomi](README_fi.md) · [ 🇩🇰 Dansk](README_da.md) · [ 🇳🇴 Norsk](README_no.md)

# 🚀 llama.cpp TurboQuant Limit Breaker (Готовий до Gemma 4 та Qwopus)

> **Порушення фізичних законів 24 ГБ VRAM: запуск Gemma-4-31B / Qwopus-27B зі 100% повним розвантаженням на GPU на одній RTX 3090 при збереженні величезного контексту 96K - 256K.**

## 🔍 Чому цей проект?

Станом на квітень 2026 року локальне розгортання спільноти штучного інтелекту з відкритим вихідним кодом зіткнулося з «глухим кутом»:
1. **Офіційне відставання**: офіційна основна гілка `llama.cpp` не інтегрувала оператори TurboQuant (`tbqp3_0`) в CUDA нативно, що призводить до миттєвого вичерпання 24 ГБ VRAM моделями 30B+ під час попереднього заповнення (prefill) довгих текстів.
2. **Проблеми сумісності форків спільноти**: існуючі гілки TurboQuant від спільноти оновлюються повільно і несумісні з новітньою архітектурою гібридного кешу SWA (Sliding Window) / ISWA, представленою в Gemma 4, що призводить до помилок `GGML_ASSERT` або помилок сегментації.
3. **Мертва петля планування агентів**: при використанні з AI IDE, такими як Claude Code, ZeroClaw або Google Antigravity, затримка prefill наддовгих промптів легко викликає повторні спроби клієнта по таймауту, через що сервер потрапляє в мертву петлю високого енергоспоживання в простої з гарячковими скасуваннями завдань (`cancel task`).

**Цей проект надає очищений патч граничних можливостей (`llama_turboquant.patch`).** Він точно впроваджує основні оператори ротаційного стиснення TurboQuant в останній офіційний код основної гілки, вручну вирішуючи конфлікти вихідного коду C++ та помилки вирівнювання пам'яті, ідеально обходячи критичні точки збоїв.

---

## 📊 Екстремальні показники продуктивності

- **Обладнання**: одна NVIDIA GeForce RTX 3090 (24 ГБ VRAM)
- **Моделі**: `Gemma-4-31B-It-UD-IQ3_XXS.gguf` / `Qwopus3.5-27B-v3-Q4_K_M.gguf`
- **Оптимізація VRAM**: 
  - Ініціалізовано контекст 98 304 (96K), використання VRAM зафіксовано на рівні **23 644 МіБ / 24 576 МіБ**.
  - KV Cache (K+V) стиснуто з >6 ГБ до **~1,48 ГБ**.
- **Швидкість генерації (Decode)**: повне прискорення `-ngl 99`, стабільне виведення на рівні **~16,5 токенів/с**.

---

## 🛠️ Посібник зі збирання та впровадження

Щоб патч застосувався ідеально і щоб уникнути помилок компіляції через зміни в апстрім API, **будь ласка, суворо зафіксуйте свій репозиторій на протестованій конкретній версії коміту**.

### 1. Підготовка репозиторію та фіксація версії
```bash
git clone https://github.com/ggml-org/llama.cpp.git
cd llama.cpp

# ОБОВ'ЯЗКОВО зафіксуйте цю безпечну версію (Build b8728)
git checkout 5e9c63546

# Застосуйте патч та зберіть
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
