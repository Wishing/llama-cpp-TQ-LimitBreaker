[ 🇺🇸 English](README.md) · [ 🇨🇳 简体中文](README_cn.md) · [ 🇯🇵 日本語](README_ja.md) · [ 🇰🇷 한국어](README_ko.md) · [ 🇻🇳 Tiếng Việt](README_vi.md) · [ 🇵🇭 Tagalog](README_tl.md) · [ 🇪🇸 Español](README_es.md) · [ 🇵🇹 Português](README_pt.md) · [ 🇮🇹 Italiano](README_it.md) · [ 🇩🇪 Deutsch](README_de.md) · [ 🇫🇷 Français](README_fr.md) · [ 🇸🇦 العربية](README_ar.md) · [ 🇮🇳 हिन्दी](README_hi.md) · [ 🇷🇺 Русский](README_ru.md) · [ 🇧🇩 বাংলা](README_bn.md) · [ 🇮🇱 עברית](README_he.md) · [ 🇵🇱 Polski](README_pl.md) · [ 🇨🇿 Čeština](README_cs.md) · [ 🇳🇱 Nederlands](README_nl.md) · [ 🇹🇷 Türkçe](README_tr.md) · [ 🇺🇦 Українська](README_uk.md) · [ 🇮🇩 Bahasa Indonesia](README_id.md) · [ 🇹🇭 ไทย](README_th.md) · [ 🇵🇰 اردو](README_ur.md) · [ 🇷🇴 Română](README_ro.md) · [ 🇸🇪 Svenska](README_sv.md) · [ 🇬🇷 Ελληνικά](README_el.md) · [ 🇭🇺 Magyar](README_hu.md) · [ 🇫🇮 Suomi](README_fi.md) · [ 🇩🇰 Dansk](README_da.md) · [ 🇳🇴 Norsk](README_no.md)

# 🚀 llama.cpp TurboQuant Limit Breaker (พร้อมสำหรับ Gemma 4 & Qwopus)

> **ทลายกฎทางฟิสิกส์ของ VRAM 24GB: รัน Gemma-4-31B / Qwopus-27B พร้อม GPU full offload 100% บน RTX 3090 เพียงใบเดียว ในขณะที่รักษา context ขนาดมหึมา 96K - 256K**

## 🔍 ทำไมต้องมีโปรเจกต์นี้?

ณ เดือนเมษายน 2026 การใช้งาน AI แบบโอเพนซอร์สในเครื่องคอมพิวเตอร์ส่วนบุคคลกำลังเผชิญกับ "ทางตัน":
1. **ความล่าช้าอย่างเป็นทางการ**: โปรเจกต์ `llama.cpp` สายหลักยังไม่ได้รวมตัวดำเนินการ TurboQuant (`tbqp3_0`) เข้ากับ CUDA อย่างเป็นทางการ ทำให้โมเดลขนาด 30B+ กิน VRAM 24GB จนหมดทันทีในระหว่างการประมวลผลข้อความยาว (prefill)
2. **ปัญหาความเข้ากันได้ของ Community Fork**: กิ่ง (branch) ของ TurboQuant ที่พัฒนาโดยชุมชนมีการอัปเดตที่ล่าช้าและไม่สามารถทำงานร่วมกับสถาปัตยกรรมแคชไฮบริด SWA (Sliding Window) / ISWA ล่าสุดที่เปิดตัวใน Gemma 4 ได้ ซึ่งนำไปสู่ข้อผิดพลาด `GGML_ASSERT` หรือการเข้าถึงหน่วยความจำผิดพลาด (segmentation faults)
3. **Agent Scheduling Death Loop**: เมื่อใช้ร่วมกับ AI IDE เช่น Claude Code, ZeroClaw หรือ Google Antigravity ความล่าช้าในการ prefill ของ prompt ที่ยาวมากจะกระตุ้นให้ฝั่งไคลเอนต์พยายามส่งคำสั่งซ้ำเนื่องจากหมดเวลา (timeout) ส่งผลให้เซิร์ฟเวอร์ตกอยู่ในวังวนมรณะของการยกเลิกงาน (`cancel task`) ซ้ำๆ ไปมาในขณะที่ใช้พลังงานสูง

**โปรเจกต์นี้ขอมอบแพตช์ขีดจำกัดที่ได้รับการขัดเกลาแล้ว (`llama_turboquant.patch`)** ซึ่งจะทำการฉีดตัวดำเนินการบีบอัด TurboQuant rotation เข้าสู่รหัสหลัก (mainline) ล่าสุดอย่างแม่นยำ โดยแก้ไขข้อขัดแย้งของซอร์สโค้ด C++ และข้อผิดพลาดในการจัดเรียงหน่วยความจำด้วยตนเอง เพื่อก้าวข้ามเส้นแดงของอาการแครชได้อย่างสมบูรณ์แบบ

---

## 📊 เกณฑ์มาตรฐานประสิทธิภาพขั้นสุดยอด

- **ฮาร์ดแวร์**: NVIDIA GeForce RTX 3090 หนึ่งใบ (24GB VRAM)
- **โมเดล**: `Gemma-4-31B-It-UD-IQ3_XXS.gguf` / `Qwopus3.5-27B-v3-Q4_K_M.gguf`
- **การเพิ่มประสิทธิภาพ VRAM**: 
  - เริ่มต้นใช้งาน context ขนาด 98,304 (96K) โดยมีการล็อคการใช้งาน VRAM ไว้ที่ **23,644 MiB / 24,576 MiB**
  - KV Cache (K+V) ถูกบีบอัดจากเดิม >6GB เหลือเพียง **~1.48 GB**
- **ความเร็วในการสร้างข้อความ (Decode)**: เร่งความเร็วเต็มสูบด้วย `-ngl 99` ให้ผลลัพธ์ที่เสถียรที่ประมาณ **~16.5 tokens/s**

---

## 🛠️ คู่มือการสร้างและการติดตั้งแพตช์

เพื่อให้แน่ใจว่าแพตช์จะถูกนำไปใช้อย่างสมบูรณ์แบบและเพื่อหลีกเลี่ยงข้อผิดพลาดในการคอมไพล์จากการเปลี่ยนแปลง API ต้นทาง **โปรดล็อค repository ของคุณไว้ที่เวอร์ชัน commit ที่ผ่านการทดสอบแล้วเท่านั้น**

### 1. เตรียม Repository และล็อคเวอร์ชัน
```bash
git clone https://github.com/ggml-org/llama.cpp.git
cd llama.cpp

# ต้องล็อคไว้ที่เวอร์ชันที่ปลอดภัยนี้ (Build b8728)
git checkout 5e9c63546

# ใช้แพตช์และสร้างโปรแกรม
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
