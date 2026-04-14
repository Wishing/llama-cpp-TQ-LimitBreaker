[ 🇺🇸 English](README.md) · [ 🇨🇳 简体中文](README_cn.md) · [ 🇯🇵 日本語](README_ja.md) · [ 🇰🇷 한국어](README_ko.md) · [ 🇻🇳 Tiếng Việt](README_vi.md) · [ 🇵🇭 Tagalog](README_tl.md) · [ 🇪🇸 Español](README_es.md) · [ 🇵🇹 Português](README_pt.md) · [ 🇮🇹 Italiano](README_it.md) · [ 🇩🇪 Deutsch](README_de.md) · [ 🇫🇷 Français](README_fr.md) · [ 🇸🇦 العربية](README_ar.md) · [ 🇮🇳 हिन्दी](README_hi.md) · [ 🇷🇺 Русский](README_ru.md) · [ 🇧🇩 বাংলা](README_bn.md) · [ 🇮🇱 עברית](README_he.md) · [ 🇵🇱 Polski](README_pl.md) · [ 🇨🇿 Čeština](README_cs.md) · [ 🇳🇱 Nederlands](README_nl.md) · [ 🇹🇷 Türkçe](README_tr.md) · [ 🇺🇦 Українська](README_uk.md) · [ 🇮🇩 Bahasa Indonesia](README_id.md) · [ 🇹🇭 ไทย](README_th.md) · [ 🇵🇰 اردو](README_ur.md) · [ 🇷🇴 Română](README_ro.md) · [ 🇸🇪 Svenska](README_sv.md) · [ 🇬🇷 Ελληνικά](README_el.md) · [ 🇭🇺 Magyar](README_hu.md) · [ 🇫🇮 Suomi](README_fi.md) · [ 🇩🇰 Dansk](README_da.md) · [ 🇳🇴 Norsk](README_no.md)

# 🚀 llama.cpp TurboQuant Limit Breaker (Pronto para Gemma 4 & Qwopus)

> **Quebrando as leis físicas de 24GB VRAM: Execute Gemma-4-31B / Qwopus-27B com 100% GPU full offload em uma única RTX 3090, mantendo um contexto massivo de 96K - 256K.**

## 🔍 Por que este projeto?

A partir de abril de 2026, a implantação local da comunidade de IA de código aberto enfrenta um "impasse":
1. **Atraso Oficial**: A linha principal oficial do `llama.cpp` não integrou nativamente os operadores TurboQuant (`tbqp3_0`) no CUDA, fazendo com que modelos de 30B+ esgotem instantaneamente os 24GB de VRAM durante o prefill de textos longos.
2. **Problemas de Compatibilidade de Forks da Comunidade**: As ramificações TurboQuant existentes na comunidade demoram a atualizar e não são compatíveis com a mais recente arquitetura de cache híbrido SWA (Sliding Window) / ISWA introduzida no Gemma 4, levando a `GGML_ASSERT` ou falhas de segmentação.
3. **Loop de Morte no Agendamento de Agentes**: Quando emparelhado com IDEs de IA como Claude Code, ZeroClaw ou Google Antigravity, a latência de prefill de prompts ultra-longos aciona facilmente tentativas de timeout do cliente, fazendo com que o servidor caia em um loop de morte de marcha lenta de alta potência de `cancel task` frenético.

**Este projeto fornece um patch de limite purificado (`llama_turboquant.patch`).** Ele injeta precisamente os operadores principais de compressão de rotação TurboQuant no código oficial mais recente, resolvendo manualmente conflitos de código-fonte C++ e erros de alinhamento de memória, ignorando perfeitamente as linhas vermelhas de travamento.

---

## 📊 Benchmarks de Desempenho Extremo

- **Hardware**: Única NVIDIA GeForce RTX 3090 (24GB VRAM)
- **Modelos**: `Gemma-4-31B-It-UD-IQ3_XXS.gguf` / `Qwopus3.5-27B-v3-Q4_K_M.gguf`
- **Otimização de VRAM**: 
  - Contexto de 98.304 (96K) inicializado, uso de VRAM bloqueado em **23.644 MiB / 24.576 MiB**.
  - KV Cache (K+V) comprimido de >6GB para **~1.48 GB**.
- **Velocidade de Geração (Decode)**: Aceleração total `-ngl 99`, saída estável em **~16.5 tokens/s**.

---

## 🛠️ Guia de Construção e Injeção

Para garantir que o patch seja aplicado perfeitamente e para evitar erros de compilação de mudanças na API upstream, **por favor, bloqueie estritamente seu repositório na versão específica de commit testada**.

### 1. Preparar Repositório & Bloquear Versão
```bash
git clone https://github.com/ggml-org/llama.cpp.git
cd llama.cpp

# DEVE bloquear nesta versão segura (Build b8728)
git checkout 5e9c63546

# Aplicar patch e build
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
