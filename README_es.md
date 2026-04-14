[ 🇺🇸 English](README.md) · [ 🇨🇳 简体中文](README_cn.md) · [ 🇯🇵 日本語](README_ja.md) · [ 🇰🇷 한국어](README_ko.md) · [ 🇻🇳 Tiếng Việt](README_vi.md) · [ 🇵🇭 Tagalog](README_tl.md) · [ 🇪🇸 Español](README_es.md) · [ 🇵🇹 Português](README_pt.md) · [ 🇮🇹 Italiano](README_it.md) · [ 🇩🇪 Deutsch](README_de.md) · [ 🇫🇷 Français](README_fr.md) · [ 🇸🇦 العربية](README_ar.md) · [ 🇮🇳 हिन्दी](README_hi.md) · [ 🇷🇺 Русский](README_ru.md) · [ 🇧🇩 বাংলা](README_bn.md) · [ 🇮🇱 עברית](README_he.md) · [ 🇵🇱 Polski](README_pl.md) · [ 🇨🇿 Čeština](README_cs.md) · [ 🇳🇱 Nederlands](README_nl.md) · [ 🇹🇷 Türkçe](README_tr.md) · [ 🇺🇦 Українська](README_uk.md) · [ 🇮🇩 Bahasa Indonesia](README_id.md) · [ 🇹🇭 ไทย](README_th.md) · [ 🇵🇰 اردو](README_ur.md) · [ 🇷🇴 Română](README_ro.md) · [ 🇸🇪 Svenska](README_sv.md) · [ 🇬🇷 Ελληνικά](README_el.md) · [ 🇭🇺 Magyar](README_hu.md) · [ 🇫🇮 Suomi](README_fi.md) · [ 🇩🇰 Dansk](README_da.md) · [ 🇳🇴 Norsk](README_no.md)

# 🚀 llama.cpp TurboQuant Limit Breaker (Preparado para Gemma 4 y Qwopus)

> **Rompiendo las leyes de la física de los 24 GB de VRAM: Ejecute Gemma-4-31B / Qwopus-27B con descarga completa (full offload) al 100% en GPU en una sola RTX 3090, manteniendo un contexto masivo de 96K - 256K.**

## 🔍 ¿Por qué este proyecto?

A partir de abril de 2026, el despliegue local de la comunidad de IA de código abierto se enfrenta a un "punto muerto":
1. **Retraso oficial**: La rama principal (mainline) oficial de `llama.cpp` no ha integrado de forma nativa los operadores de TurboQuant (`tbqp3_0`) en CUDA, lo que provoca que los modelos de más de 30B agoten instantáneamente los 24 GB de VRAM durante el pre-procesamiento (prefill) de textos largos.
2. **Problemas de compatibilidad con forks de la comunidad**: Las ramas de TurboQuant existentes en la comunidad tardan en actualizarse y no son compatibles con la última arquitectura de caché híbrida SWA (Sliding Window) / ISWA introducida en Gemma 4, lo que provoca errores `GGML_ASSERT` o fallos de segmentación.
3. **Bucle de muerte en la programación de agentes**: Cuando se combina con IDEs de IA como Claude Code, ZeroClaw o Google Antigravity, la latencia de pre-procesamiento de prompts ultra largos activa fácilmente reintentos por tiempo de espera del cliente, lo que hace que el servidor caiga en un bucle de muerte de alta potencia en reposo cancelando tareas frenéticamente.

**Este proyecto proporciona un parche de límite purificado (`llama_turboquant.patch`).** Inyecta con precisión los operadores principales de compresión por rotación de TurboQuant en el código de la rama principal oficial más reciente, resolviendo manualmente los conflictos de código fuente de C++ y los errores de alineación de memoria, evitando perfectamente las líneas rojas de cuelgues.

---

## 📊 Pruebas de rendimiento extremo

- **Hardware**: Una sola NVIDIA GeForce RTX 3090 (24 GB de VRAM)
- **Modelos**: `Gemma-4-31B-It-UD-IQ3_XXS.gguf` / `Qwopus3.5-27B-v3-Q4_K_M.gguf`
- **Optimización de VRAM**: 
  - Contexto de 98.304 (96K) inicializado, uso de VRAM bloqueado en **23.644 MiB / 24.576 MiB**.
  - KV Cache (K+V) comprimido de >6 GB a **~1,48 GB**.
- **Velocidad de generación (Decode)**: Aceleración `-ngl 99` completa, salida estable a **~16,5 tokens/s**.

---

## 🛠️ Guía de construcción e inyección

Para asegurar que el parche se aplique perfectamente y evitar errores de compilación por cambios en la API ascendente (upstream), **por favor, bloquee estrictamente su repositorio en la versión de commit específica probada**.

### 1. Preparar el repositorio y bloquear la versión
```bash
git clone https://github.com/ggml-org/llama.cpp.git
cd llama.cpp

# DEBE bloquearse en esta versión segura (Build b8728)
git checkout 5e9c63546

# Aplicar parche y construir
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
