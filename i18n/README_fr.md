[ 🇺🇸 English](../README.md) · [ 🇨🇳 简体中文](README_cn.md) · [ 🇯🇵 日本語](README_ja.md) · [ 🇰🇷 한국어](README_ko.md) · [ 🇻🇳 Tiếng Việt](README_vi.md) · [ 🇵🇭 Tagalog](README_tl.md) · [ 🇪🇸 Español](README_es.md) · [ 🇵🇹 Português](README_pt.md) · [ 🇮🇹 Italiano](README_it.md) · [ 🇩🇪 Deutsch](README_de.md) · [ 🇫🇷 Français](README_fr.md) · [ 🇸🇦 العربية](README_ar.md) · [ 🇮🇳 हिन्दी](README_hi.md) · [ 🇷🇺 Русский](README_ru.md) · [ 🇧🇩 বাংলা](README_bn.md) · [ 🇮🇱 עברית](README_he.md) · [ 🇵🇱 Polski](README_pl.md) · [ 🇨🇿 Čeština](README_cs.md) · [ 🇳🇱 Nederlands](README_nl.md) · [ 🇹🇷 Türkçe](README_tr.md) · [ 🇺🇦 Українська](README_uk.md) · [ 🇮🇩 Bahasa Indonesia](README_id.md) · [ 🇹🇭 ไทย](README_th.md) · [ 🇵🇰 اردو](README_ur.md) · [ 🇷🇴 Română](README_ro.md) · [ 🇸🇪 Svenska](README_sv.md) · [ 🇬🇷 Ελληνικά](README_el.md) · [ 🇭🇺 Magyar](README_hu.md) · [ 🇫🇮 Suomi](README_fi.md) · [ 🇩🇰 Dansk](README_da.md) · [ 🇳🇴 Norsk](README_no.md)

# 🚀 llama.cpp TurboQuant Limit Breaker (Prêt pour Gemma 4 & Qwopus)

> **Briser les lois physiques de 24 Go de VRAM : Exécutez Gemma-4-31B / Qwopus-27B avec un déchargement GPU complet à 100 % sur une seule RTX 3090, tout en conservant un contexte massif de 96K à 256K.**

## 🔍 Pourquoi ce projet ?

En date d'avril 2026, le déploiement local de la communauté AI open-source fait face à une « impasse » :
1. **Retard officiel** : La branche principale officielle de `llama.cpp` n'a pas intégré nativement les opérateurs TurboQuant (`tbqp3_0`) dans CUDA, ce qui fait que les modèles de plus de 30B épuisent instantanément les 24 Go de VRAM lors du pré-remplissage (prefill) de textes longs.
2. **Problèmes de compatibilité des forks communautaires** : Les branches TurboQuant existantes de la communauté sont lentes à se mettre à jour et ne sont pas compatibles avec la dernière architecture de cache hybride SWA (Sliding Window) / ISWA introduite dans Gemma 4, ce qui entraîne des erreurs `GGML_ASSERT` ou des fautes de segmentation.
3. **Boucle infernale de planification d'agent** : Lorsqu'il est couplé à des IDE d'IA comme Claude Code, ZeroClaw ou Google Antigravity, la latence de prefill de prompts ultra-longs déclenche facilement des tentatives de timeout du client, faisant tomber le serveur dans une boucle infernale de consommation élevée au repos avec des `cancel task` frénétiques.

**Ce projet fournit un correctif de limite purifié (`llama_turboquant.patch`).** Il injecte précisément les opérateurs de compression de rotation TurboQuant de base dans le dernier code officiel, résolvant manuellement les conflits de code source C++ et les erreurs d'alignement de la mémoire, contournant parfaitement les lignes rouges de plantage.

---

## 📊 Benchmarks de performances extrêmes

- **Matériel** : Une seule NVIDIA GeForce RTX 3090 (24 Go de VRAM)
- **Modèles** : `Gemma-4-31B-It-UD-IQ3_XXS.gguf` / `Qwopus3.5-27B-v3-Q4_K_M.gguf`
- **Optimisation de la VRAM** : 
  - Contexte de 98 304 (96K) initialisé, utilisation de la VRAM verrouillée à **23 644 MiB / 24 576 MiB**.
  - KV Cache (K+V) compressé de >6 Go à **~1,48 Go**.
- **Vitesse de génération (Decode)** : Accélération complète `-ngl 99`, sortie stable à **~16,5 tokens/s**.

---

## 🛠️ Guide de construction et d'injection

Pour garantir que le patch est appliqué parfaitement et pour éviter les erreurs de compilation dues aux changements d'API en amont, **veuillez verrouiller strictement votre dépôt à la version spécifique du commit testé**.

### 1. Préparer le dépôt et verrouiller la version
```bash
git clone https://github.com/ggml-org/llama.cpp.git
cd llama.cpp

# DOIT être verrouillé sur cette version sûre (Build b8728)
git checkout 5e9c63546

# Appliquer le patch et construire
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
