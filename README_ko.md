[ 🇺🇸 English](README.md) · [ 🇨🇳 简体中文](README_cn.md) · [ 🇯🇵 日本語](README_ja.md) · [ 🇰🇷 한국어](README_ko.md) · [ 🇻🇳 Tiếng Việt](README_vi.md) · [ 🇵🇭 Tagalog](README_tl.md) · [ 🇪🇸 Español](README_es.md) · [ 🇵🇹 Português](README_pt.md) · [ 🇮🇹 Italiano](README_it.md) · [ 🇩🇪 Deutsch](README_de.md) · [ 🇫🇷 Français](README_fr.md) · [ 🇸🇦 العربية](README_ar.md) · [ 🇮🇳 हिन्दी](README_hi.md) · [ 🇷🇺 Русский](README_ru.md) · [ 🇧🇩 বাংলা](README_bn.md) · [ 🇮🇱 עברית](README_he.md) · [ 🇵🇱 Polski](README_pl.md) · [ 🇨🇿 Čeština](README_cs.md) · [ 🇳🇱 Nederlands](README_nl.md) · [ 🇹🇷 Türkçe](README_tr.md) · [ 🇺🇦 Українська](README_uk.md) · [ 🇮🇩 Bahasa Indonesia](README_id.md) · [ 🇹🇭 ไทย](README_th.md) · [ 🇵🇰 اردو](README_ur.md) · [ 🇷🇴 Română](README_ro.md) · [ 🇸🇪 Svenska](README_sv.md) · [ 🇬🇷 Ελληνικά](README_el.md) · [ 🇭🇺 Magyar](README_hu.md) · [ 🇫🇮 Suomi](README_fi.md) · [ 🇩🇰 Dansk](README_da.md) · [ 🇳🇴 Norsk](README_no.md)

# 🚀 llama.cpp TurboQuant Limit Breaker (Gemma 4 & Qwopus 준비 완료)

> **24GB VRAM의 물리 법칙 파괴: 단일 RTX 3090에서 96K - 256K의 거대 컨텍스트를 유지하면서 Gemma-4-31B / Qwopus-27B를 100% GPU 풀 오프로드로 실행.**

## 🔍 왜 이 프로젝트가 필요한가요?

2026년 4월 현재, 오픈소스 AI 커뮤니티의 로컬 배포는 "교착 상태"에 직면해 있습니다:
1. **공식 지원 지연**: 공식 `llama.cpp` 메인라인에 TurboQuant 연산자(`tbqp3_0`)가 CUDA에 기본적으로 통합되지 않아, 30B 이상의 모델이 긴 텍스트 프리필(prefill) 중에 24GB VRAM을 즉시 소진하게 만듭니다.
2. **커뮤니티 포크 호환성 문제**: 기존 커뮤니티 TurboQuant 브랜치들은 업데이트가 느려 Gemma 4에서 도입된 최신 SWA(슬라이딩 윈도우) / ISWA 하이브리드 캐시 아키텍처와 호환되지 않으며, 이로 인해 `GGML_ASSERT` 또는 세그멘테이션 오류(segmentation fault)가 발생합니다.
3. **에이전트 스케줄링 데스 루프**: Claude Code, ZeroClaw 또는 Google Antigravity와 같은 AI IDE와 함께 사용할 때, 초장문 프롬프트의 프리필 지연이 클라이언트 타임아웃 재시도를 유발하여 서버가 광란의 `cancel task`를 반복하는 고전력 유휴 데스 루프에 빠지게 합니다.

**이 프로젝트는 정제된 리미트 패치(`llama_turboquant.patch`)를 제공합니다.** 코어 TurboQuant 회전 압축 연산자를 최신 공식 메인라인 코드에 정밀하게 주입하여 C++ 소스 충돌 및 메모리 정렬 오류를 수동으로 해결하고 충돌 레드라인을 완벽하게 우회합니다.

---

## 📊 극한 성능 벤치마크

- **하드웨어**: 단일 NVIDIA GeForce RTX 3090 (24GB VRAM)
- **모델**: `Gemma-4-31B-It-UD-IQ3_XXS.gguf` / `Qwopus3.5-27B-v3-Q4_K_M.gguf`
- **VRAM 최적화**: 
  - 98,304 (96K) 컨텍스트 초기화 시 VRAM 사용량이 **23,644 MiB / 24,576 MiB**로 고정됨.
  - KV 캐시(K+V)가 6GB 이상에서 **~1.48 GB**로 압축됨.
- **생성 속도 (Decode)**: 전체 `-ngl 99` 가속을 통해 **~16.5 tokens/s**의 안정적인 출력.

---

## 🛠️ 빌드 및 주입 가이드

패치가 완벽하게 적용되고 업스트림 API 변경으로 인한 컴파일 오류를 방지하려면 **반드시 리포지토리를 테스트된 특정 커밋 버전으로 고정하십시오**.

### 1. 리포지토리 준비 및 버전 고정
```bash
git clone https://github.com/ggml-org/llama.cpp.git
cd llama.cpp

# 반드시 이 안전한 버전으로 고정해야 합니다 (Build b8728)
git checkout 5e9c63546

# 패치 적용 및 빌드
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
