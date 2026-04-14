[ 🇺🇸 English](../README.md) · [ 🇨🇳 简体中文](README_cn.md) · [ 🇯🇵 日本語](README_ja.md) · [ 🇰🇷 한국어](README_ko.md) · [ 🇻🇳 Tiếng Việt](README_vi.md) · [ 🇵🇭 Tagalog](README_tl.md) · [ 🇪🇸 Español](README_es.md) · [ 🇵🇹 Português](README_pt.md) · [ 🇮🇹 Italiano](README_it.md) · [ 🇩🇪 Deutsch](README_de.md) · [ 🇫🇷 Français](README_fr.md) · [ 🇸🇦 العربية](README_ar.md) · [ 🇮🇳 हिन्दी](README_hi.md) · [ 🇷🇺 Русский](README_ru.md) · [ 🇧🇩 বাংলা](README_bn.md) · [ 🇮🇱 עברית](README_he.md) · [ 🇵🇱 Polski](README_pl.md) · [ 🇨🇿 Čeština](README_cs.md) · [ 🇳🇱 Nederlands](README_nl.md) · [ 🇹🇷 Türkçe](README_tr.md) · [ 🇺🇦 Українська](README_uk.md) · [ 🇮🇩 Bahasa Indonesia](README_id.md) · [ 🇹🇭 ไทย](README_th.md) · [ 🇵🇰 اردو](README_ur.md) · [ 🇷🇴 Română](README_ro.md) · [ 🇸🇪 Svenska](README_sv.md) · [ 🇬🇷 Ελληνικά](README_el.md) · [ 🇭🇺 Magyar](README_hu.md) · [ 🇫🇮 Suomi](README_fi.md) · [ 🇩🇰 Dansk](README_da.md) · [ 🇳🇴 Norsk](README_no.md)

# 🚀 llama.cpp TurboQuant Limit Breaker (Sẵn sàng cho Gemma 4 & Qwopus)

> **Phá vỡ các quy luật vật lý của 24GB VRAM: Chạy Gemma-4-31B / Qwopus-27B với 100% GPU full offload trên một card RTX 3090 duy nhất, trong khi vẫn duy trì context khổng lồ từ 96K - 256K.**

## 🔍 Tại sao có dự án này?

Tính đến tháng 4 năm 2026, việc triển khai cục bộ cộng đồng AI mã nguồn mở đang đối mặt với một "thế bế tắc":
1. **Sự chậm trễ chính thức**: Nhánh chính thức của `llama.cpp` vẫn chưa tích hợp các toán tử TurboQuant (`tbqp3_0`) vào CUDA một cách tự nhiên, khiến các mô hình trên 30B ngay lập tức làm cạn kiệt 24GB VRAM trong quá trình prefill văn bản dài.
2. **Vấn đề tương thích của các nhánh cộng đồng**: Các nhánh TurboQuant hiện có của cộng đồng cập nhật chậm và không tương thích với kiến trúc cache hybrid SWA (Sliding Window) / ISWA mới nhất được giới thiệu trong Gemma 4, dẫn đến lỗi `GGML_ASSERT` hoặc lỗi phân đoạn (segmentation faults).
3. **Vòng lặp tử thần trong điều phối Agent**: Khi kết hợp với các AI IDE như Claude Code, ZeroClaw hoặc Google Antigravity, độ trễ prefill của các prompt cực dài dễ dàng kích hoạt việc thử lại do quá hạn (timeout) từ phía client, khiến server rơi vào vòng lặp tử thần tiêu thụ điện năng cao nhưng không hiệu quả vì liên tục `cancel task`.

**Dự án này cung cấp một bản vá giới hạn tinh khiết (`llama_turboquant.patch`).** Nó tiêm chính xác các toán tử nén xoay TurboQuant cốt lõi vào mã nguồn nhánh chính mới nhất, giải quyết thủ công các xung đột mã nguồn C++ và lỗi căn chỉnh bộ nhớ, vượt qua các giới hạn gây crash một cách hoàn hảo.

---

## 📊 Chỉ số hiệu suất cực hạn

- **Phần cứng**: Một card NVIDIA GeForce RTX 3090 (24GB VRAM)
- **Mô hình**: `Gemma-4-31B-It-UD-IQ3_XXS.gguf` / `Qwopus3.5-27B-v3-Q4_K_M.gguf`
- **Tối ưu hóa VRAM**: 
  - Khởi tạo context 98,304 (96K), mức sử dụng VRAM được khóa ở mức **23,644 MiB / 24,576 MiB**.
  - KV Cache (K+V) được nén từ >6GB xuống còn **~1.48 GB**.
- **Tốc độ tạo (Decode)**: Tăng tốc full `-ngl 99`, đầu ra ổn định ở mức **~16.5 tokens/s**.

---

## 🛠️ Hướng dẫn Build & Tiêm bản vá

Để đảm bảo bản vá được áp dụng hoàn hảo và tránh các lỗi biên dịch do thay đổi API từ thượng nguồn, **vui lòng khóa chặt kho lưu trữ của bạn ở phiên bản commit cụ thể đã được kiểm tra**.

### 1. Chuẩn bị kho lưu trữ & Khóa phiên bản
```bash
git clone https://github.com/ggml-org/llama.cpp.git
cd llama.cpp

# PHẢI khóa ở phiên bản an toàn này (Build b8728)
git checkout 5e9c63546

# Áp dụng bản vá và build
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
