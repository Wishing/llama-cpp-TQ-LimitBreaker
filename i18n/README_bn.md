[ 🇺🇸 English](../README.md) · [ 🇨🇳 简体中文](README_cn.md) · [ 🇯🇵 日本語](README_ja.md) · [ 🇰🇷 한국어](README_ko.md) · [ 🇻🇳 Tiếng Việt](README_vi.md) · [ 🇵🇭 Tagalog](README_tl.md) · [ 🇪🇸 Español](README_es.md) · [ 🇵🇹 Português](README_pt.md) · [ 🇮🇹 Italiano](README_it.md) · [ 🇩🇪 Deutsch](README_de.md) · [ 🇫🇷 Français](README_fr.md) · [ 🇸🇦 العربية](README_ar.md) · [ 🇮🇳 हिन्दी](README_hi.md) · [ 🇷🇺 Русский](README_ru.md) · [ 🇧🇩 বাংলা](README_bn.md) · [ 🇮🇱 עברית](README_he.md) · [ 🇵🇱 Polski](README_pl.md) · [ 🇨🇿 Čeština](README_cs.md) · [ 🇳🇱 Nederlands](README_nl.md) · [ 🇹🇷 Türkçe](README_tr.md) · [ 🇺🇦 Українська](README_uk.md) · [ 🇮🇩 Bahasa Indonesia](README_id.md) · [ 🇹🇭 ไทย](README_th.md) · [ 🇵🇰 اردو](README_ur.md) · [ 🇷🇴 Română](README_ro.md) · [ 🇸🇪 Svenska](README_sv.md) · [ 🇬🇷 Ελληνικά](README_el.md) · [ 🇭🇺 Magyar](README_hu.md) · [ 🇫🇮 Suomi](README_fi.md) · [ 🇩🇰 Dansk](README_da.md) · [ 🇳🇴 Norsk](README_no.md)

# 🚀 llama.cpp TurboQuant Limit Breaker (Gemma 4 এবং Qwopus এর জন্য প্রস্তুত)

> **24GB VRAM এর পদার্থবিজ্ঞানের সূত্র ভাঙা: একটি সিঙ্গেল RTX 3090 তে 100% GPU ফুল অফলোড সহ Gemma-4-31B / Qwopus-27B চালান, এবং 96K - 256K বিশাল কনটেক্সট বজায় রাখুন।**

## 🔍 কেন এই প্রকল্প?

২০২৬ সালের এপ্রিল পর্যন্ত, ওপেন-সোর্স AI সম্প্রদায়ের স্থানীয় স্থাপনা একটি "অচলাবস্থার" সম্মুখীন হচ্ছে:
১. **আফিসিয়াল ল্যাগ**: অফিসিয়াল `llama.cpp` মেইনলাইনে TurboQuant অপারেটরগুলো (`tbqp3_0`) নেটিভভাবে CUDA তে যুক্ত করা হয়নি, যার ফলে দীর্ঘ টেক্সট প্রিফিলের সময় 30B+ মডেলগুলো তাৎক্ষণিকভাবে 24GB VRAM শেষ করে ফেলে।
২. **কমিউনিটি ফোর্ক সামঞ্জস্যতা সমস্যা**: বর্তমান কমিউনিটি TurboQuant ব্রাঞ্চগুলো আপডেট করতে ধীর এবং Gemma 4 এ প্রবর্তিত সর্বশেষ SWA (Sliding Window) / ISWA হাইব্রিড ক্যাশ আর্কিটেকচারের সাথে সামঞ্জস্যপূর্ণ নয়, যা `GGML_ASSERT` বা সেগমেন্টেশন ফল্ট ঘটায়।
৩. **এজেন্ট শিডিউলিং ডেথ লুপ**: যখন Claude Code, ZeroClaw বা Google Antigravity এর মতো AI IDE গুলোর সাথে ব্যবহার করা হয়, তখন অতি-দীর্ঘ প্রম্পটের প্রিফিল লেটেন্সি সহজেই ক্লায়েন্ট টাইমআউট রিট্রাই ট্রিগার করে, যার ফলে সার্ভার উন্মত্ত `cancel task` এর হাই-পাওয়ার আইডল ডেথ লুপে পড়ে যায়।

**এই প্রকল্পটি একটি পরিশোধিত লিমিট প্যাচ (`llama_turboquant.patch`) প্রদান করে।** এটি সর্বশেষ অফিসিয়াল মেইনলাইন কোডে কোর TurboQuant রোটেশন কম্প্রেশন অপারেটরগুলোকে নির্ভুলভাবে ইনজেক্ট করে, ম্যানুয়ালি C++ সোর্স কনফ্লিক্ট এবং মেমরি অ্যালাইনমেন্ট ত্রুটিগুলো সমাধান করে এবং নিখুঁতভাবে ক্র্যাশ রেড লাইনগুলোকে এড়িয়ে যায়।

---

## 📊 চরম কর্মক্ষমতা বেঞ্চমার্ক

- **হার্ডওয়্যার**: একটি সিঙ্গেল NVIDIA GeForce RTX 3090 (24GB VRAM)
- **মডেল**: `Gemma-4-31B-It-UD-IQ3_XXS.gguf` / `Qwopus3.5-27B-v3-Q4_K_M.gguf`
- **VRAM অপ্টিমাইজেশন**: 
  - 98,304 (96K) কনটেক্সট শুরু করা হয়েছে, VRAM ব্যবহার **23,644 MiB / 24,576 MiB** এ লক করা হয়েছে।
  - KV Cache (K+V) >6GB থেকে **~1.48 GB** তে সংকুচিত করা হয়েছে।
- **জেনারেশন স্পিড (Decode)**: ফুল `-ngl 99` অ্যাক্সিলারেশন, **~16.5 tokens/s** এ স্থিতিশীল আউটপুট।

---

## 🛠️ বিল্ড এবং ইনজেকশন গাইড

প্যাচটি নিখুঁতভাবে প্রয়োগ করা হয়েছে কিনা তা নিশ্চিত করতে এবং আপস্ট্রিম API পরিবর্তন থেকে কম্পাইলেশন ত্রুটি এড়াতে, **অনুগ্রহ করে আপনার রিপোজিটরিটিকে পরীক্ষিত নির্দিষ্ট কমিট সংস্করণে কঠোরভাবে লক করুন**।

### ১. রিপোজিটরি প্রস্তুত করুন এবং সংস্করণ লক করুন
```bash
git clone https://github.com/ggml-org/llama.cpp.git
cd llama.cpp

# অবশ্যই এই নিরাপদ সংস্করণে লক করতে হবে (Build b8728)
git checkout 5e9c63546

# প্যাচ প্রয়োগ করুন এবং বিল্ড করুন
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
