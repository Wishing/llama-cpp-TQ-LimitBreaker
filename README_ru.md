[ 🇺🇸 English](README.md) · [ 🇨🇳 简体中文](README_cn.md) · [ 🇯🇵 日本語](README_ja.md) · [ 🇰🇷 한국어](README_ko.md) · [ 🇻🇳 Tiếng Việt](README_vi.md) · [ 🇵🇭 Tagalog](README_tl.md) · [ 🇪🇸 Español](README_es.md) · [ 🇵🇹 Português](README_pt.md) · [ 🇮🇹 Italiano](README_it.md) · [ 🇩🇪 Deutsch](README_de.md) · [ 🇫🇷 Français](README_fr.md) · [ 🇸🇦 العربية](README_ar.md) · [ 🇮🇳 हिन्दी](README_hi.md) · [ 🇷🇺 Русский](README_ru.md) · [ 🇧🇩 বাংলা](README_bn.md) · [ 🇮🇱 עברית](README_he.md) · [ 🇵🇱 Polski](README_pl.md) · [ 🇨🇿 Čeština](README_cs.md) · [ 🇳🇱 Nederlands](README_nl.md) · [ 🇹🇷 Türkçe](README_tr.md) · [ 🇺🇦 Українська](README_uk.md) · [ 🇮🇩 Bahasa Indonesia](README_id.md) · [ 🇹🇭 ไทย](README_th.md) · [ 🇵🇰 اردو](README_ur.md) · [ 🇷🇴 Română](README_ro.md) · [ 🇸🇪 Svenska](README_sv.md) · [ 🇬🇷 Ελληνικά](README_el.md) · [ 🇭🇺 Magyar](README_hu.md) · [ 🇫🇮 Suomi](README_fi.md) · [ 🇩🇰 Dansk](README_da.md) · [ 🇳🇴 Norsk](README_no.md)

# 🚀 llama.cpp TurboQuant Limit Breaker (Готов к Gemma 4 и Qwopus)

> **Нарушение физических законов 24 ГБ VRAM: запуск Gemma-4-31B / Qwopus-27B со 100% полной разгрузкой на GPU на одной RTX 3090 при сохранении огромного контекста 96K - 256K.**

## 🔍 Почему этот проект?

По состоянию на апрель 2026 года локальное развертывание сообщества ИИ с открытым исходным кодом столкнулось с «тупиком»:
1. **Официальное отставание**: официальная основная ветка `llama.cpp` не интегрировала операторы TurboQuant (`tbqp3_0`) в CUDA нативно, что приводит к мгновенному исчерпанию 24 ГБ VRAM моделями 30B+ во время предварительного заполнения (prefill) длинных текстов.
2. **Проблемы совместимости форков сообщества**: существующие ветки TurboQuant от сообщества обновляются медленно и несовместимы с новейшей архитектурой гибридного кэша SWA (Sliding Window) / ISWA, представленной в Gemma 4, что приводит к ошибкам `GGML_ASSERT` или ошибкам сегментации.
3. **Мертвая петля планирования агентов**: при использовании с AI IDE, такими как Claude Code, ZeroClaw или Google Antigravity, задержка prefill сверхдлинных промптов легко вызывает повторные попытки клиента по таймауту, из-за чего сервер попадает в мертвую петлю высокого энергопотребления в простое с неистовыми отменами задач (`cancel task`).

**Этот проект предоставляет очищенный патч предельных возможностей (`llama_turboquant.patch`).** Он точно внедряет основные операторы ротационного сжатия TurboQuant в последний официальный код основной ветки, вручную разрешая конфликты исходного кода C++ и ошибки выравнивания памяти, идеально обходя критические точки сбоев.

---

## 📊 Экстремальные показатели производительности

- **Оборудование**: одна NVIDIA GeForce RTX 3090 (24 ГБ VRAM)
- **Модели**: `Gemma-4-31B-It-UD-IQ3_XXS.gguf` / `Qwopus3.5-27B-v3-Q4_K_M.gguf`
- **Оптимизация VRAM**: 
  - Инициализирован контекст 98 304 (96K), использование VRAM зафиксировано на уровне **23 644 МиБ / 24 576 МиБ**.
  - KV Cache (K+V) сжат с >6 ГБ до **~1,48 ГБ**.
- **Скорость генерации (Decode)**: полное ускорение `-ngl 99`, стабильный вывод на уровне **~16,5 токенов/с**.

---

## 🛠️ Руководство по сборке и внедрению

Чтобы патч применился идеально и во избежание ошибок компиляции из-за изменений в апстрим API, **пожалуйста, строго зафиксируйте свой репозиторий на протестированной конкретной версии коммита**.

### 1. Подготовка репозитория и фиксация версии
```bash
git clone https://github.com/ggml-org/llama.cpp.git
cd llama.cpp

# ОБЯЗАТЕЛЬНО зафиксируйте эту безопасную версию (Build b8728)
git checkout 5e9c63546

# Примените патч и соберите
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
