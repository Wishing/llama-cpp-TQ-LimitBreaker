[ 🇺🇸 English](README.md) · [ 🇨🇳 简体中文](README_cn.md) · [ 🇯🇵 日本語](README_ja.md) · [ 🇰🇷 한국어](README_ko.md) · [ 🇻🇳 Tiếng Việt](README_vi.md) · [ 🇵🇭 Tagalog](README_tl.md) · [ 🇪🇸 Español](README_es.md) · [ 🇵🇹 Português](README_pt.md) · [ 🇮🇹 Italiano](README_it.md) · [ 🇩🇪 Deutsch](README_de.md) · [ 🇫🇷 Français](README_fr.md) · [ 🇸🇦 العربية](README_ar.md) · [ 🇮🇳 हिन्दी](README_hi.md) · [ 🇷🇺 Русский](README_ru.md) · [ 🇧🇩 বাংলা](README_bn.md) · [ 🇮🇱 עברית](README_he.md) · [ 🇵🇱 Polski](README_pl.md) · [ 🇨🇿 Čeština](README_cs.md) · [ 🇳🇱 Nederlands](README_nl.md) · [ 🇹🇷 Türkçe](README_tr.md) · [ 🇺🇦 Українська](README_uk.md) · [ 🇮🇩 Bahasa Indonesia](README_id.md) · [ 🇹🇭 ไทย](README_th.md) · [ 🇵🇰 اردو](README_ur.md) · [ 🇷🇴 Română](README_ro.md) · [ 🇸🇪 Svenska](README_sv.md) · [ 🇬🇷 Ελληνικά](README_el.md) · [ 🇭🇺 Magyar](README_hu.md) · [ 🇫🇮 Suomi](README_fi.md) · [ 🇩🇰 Dansk](README_da.md) · [ 🇳🇴 Norsk](README_no.md)

# 🚀 llama.cpp TurboQuant Limit Breaker (Gemma 4 और Qwopus तैयार)

> **24GB VRAM के भौतिक नियमों को तोड़ना: एक ही RTX 3090 पर 100% GPU फुल ऑफलोड के साथ Gemma-4-31B / Qwopus-27B चलाएं, जबकि 96K - 256K विशाल संदर्भ बनाए रखें।**

## 🔍 यह परियोजना क्यों?

अप्रैल 2026 तक, ओपन-सोर्स AI समुदाय का स्थानीय परिनियोजन एक "गतिरोध" का सामना कर रहा है:
1. **आधिकारिक विलंब**: आधिकारिक `llama.cpp` मेनलाइन ने TurboQuant ऑपरेटरों (`tbqp3_0`) को CUDA में मूल रूप से एकीकृत नहीं किया है, जिससे 30B+ मॉडल लंबे टेक्स्ट प्रीफिल के दौरान तुरंत 24GB VRAM समाप्त कर देते हैं।
2. **सामुदायिक फोर्क संगतता मुद्दे**: मौजूदा सामुदायिक TurboQuant शाखाएं अपडेट करने में धीमी हैं और Gemma 4 में पेश किए गए नवीनतम SWA (स्लाइडिंग विंडो) / ISWA हाइब्रिड कैश आर्किटेक्चर के साथ संगत नहीं हो सकती हैं, जिससे `GGML_ASSERT` या सेगमेंटेशन फॉल्ट हो सकते हैं।
3. **एजेंट शेड्यूलिंग डेथ लूप**: जब क्लाउड कोड (Claude Code), ज़ीरोक्लॉ (ZeroClaw), या गूगल एंटीग्रेविटी (Google Antigravity) जैसे AI IDE के साथ जोड़ा जाता है, तो अल्ट्रा-लॉन्ग प्रॉम्प्ट की प्रीफिल लेटेंसी आसानी से क्लाइंट टाइमआउट रिट्राइ को ट्रिगर करती है, जिससे सर्वर उन्मत्त `cancel task` के हाई-पावर आइडल डेथ लूप में गिर जाता है।

**यह परियोजना एक शुद्ध सीमा पैच (`llama_turboquant.patch`) प्रदान करती है।** यह नवीनतम आधिकारिक मेनलाइन कोड में कोर TurboQuant रोटेशन कम्प्रेशन ऑपरेटरों को सटीक रूप से इंजेक्ट करता है, C++ स्रोत संघर्षों और मेमोरी संरेखण त्रुटियों को मैन्युअल रूप से हल करता है, पूरी तरह से क्रैश रेड लाइनों को बायपास करता है।

---

## 📊 चरम प्रदर्शन बेंचमार्क

- **हार्डवेयर**: सिंगल NVIDIA GeForce RTX 3090 (24GB VRAM)
- **मॉडल**: `Gemma-4-31B-It-UD-IQ3_XXS.gguf` / `Qwopus3.5-27B-v3-Q4_K_M.gguf`
- **VRAM अनुकूलन**: 
  - 98,304 (96K) संदर्भ प्रारंभ किया गया, VRAM उपयोग **23,644 MiB / 24,576 MiB** पर लॉक किया गया।
  - KV Cache (K+V) को >6GB से **~1.48 GB** तक संकुचित किया गया।
- **जेनरेशन स्पीड (Decode)**: पूर्ण `-ngl 99` त्वरण, **~16.5 tokens/s** पर स्थिर आउटपुट।

---

## 🛠️ बिल्ड और इंजेक्शन गाइड

यह सुनिश्चित करने के लिए कि पैच पूरी तरह से लागू किया गया है और अपस्ट्रीम API परिवर्तनों से संकलन त्रुटियों से बचने के लिए, **कृपया अपने रिपॉजिटरी को परीक्षण किए गए विशिष्ट कमिट संस्करण पर सख्ती से लॉक करें**।

### 1. रिपॉजिटरी तैयार करें और संस्करण लॉक करें
```bash
git clone https://github.com/ggml-org/llama.cpp.git
cd llama.cpp

# इस सुरक्षित संस्करण (Build b8728) पर लॉक होना चाहिए
git checkout 5e9c63546

# पैच लागू करें और बिल्ड करें
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
