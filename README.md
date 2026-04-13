# 🧮 Lecture Tracking System — Mr. Islam Rageh

> نظام متكامل لإدارة المحاضرات وتتبع حضور الطلاب وأداء الاختبارات، مبني على **Telegram Mini Apps** + **Google Sheets** + **n8n**.

---

## 📋 فهرس المحتويات

- [نظرة عامة](#نظرة-عامة)
- [مكونات المشروع](#مكونات-المشروع)
- [هيكل الملفات](#هيكل-الملفات)
- [كيف يعمل النظام — بالتفصيل](#كيف-يعمل-النظام)
- [Google Sheets — بنية البيانات](#google-sheets)
- [n8n Workflow — الـ 5 Triggers](#n8n-workflow)
- [Telegram Proxy](#telegram-proxy)
- [النشر عبر GitHub Actions](#github-actions)

---

## 🎯 نظرة عامة

المشروع ده بيحل مشكلة حقيقية: **كيف تبعت محاضرة لكل طلابك تلقائياً، وتتابع مين فتحها، ومين عمل الاختبار، ومين مش موجود؟**

الإجابة: كل ده بيحصل تلقائياً بدون أي تدخل يدوي — بمجرد ما المدرس يكتب `post` في خلية في الـ Google Sheet.

---

## 🧩 مكونات المشروع

| المكون | الغرض | التقنية |
|--------|--------|---------|
| `register.html` | تسجيل الطلاب الجدد | Telegram Mini App |
| `quiz.html` | اختبار المحاضرة | Telegram Mini App |
| **n8n Workflow** | كل الـ backend | n8n على Hugging Face |
| **Google Sheets** | قاعدة البيانات الكاملة | gviz API + Sheets API |
| **Telegram Proxy** | إرسال الرسائل | Vercel Serverless |

```
┌──────────────────────────────────────────────────────┐
│                   Google Sheets                      │
│  lectures │ Distribution │ Students │ Group Cods     │
└─────┬──────────────┬───────────────┬─────────────────┘
      │ Trigger      │ Read          │ Read/Write
      ▼              ▼               ▼
┌─────────────────────────────────────────────────────┐
│              n8n Workflow (5 Triggers)               │
│                                                     │
│  [Sheets Trigger]  →  إرسال لنك المحاضرة          │
│  [Sheets Trigger]  →  إرسال بيانات الدرس          │
│  [GET /lecture-track] → تسجيل الحضور + redirect   │
│  [GET /get-classes]   → جلب المجموعات             │
│  [POST /register-student] → حفظ بيانات الطالب     │
│  [POST /quiz-result]  → حفظ نتيجة الاختبار       │
└─────┬───────────────────────────────────────────────┘
      │ Telegram API (via Proxy)
      ▼
┌─────────────────────────────────────────────────────┐
│                  Telegram Bot                        │
│   register Mini App  │  quiz Mini App               │
└─────────────────────────────────────────────────────┘
```

---

## 📁 هيكل الملفات

```
lecture-tracking-system/
├── .github/
│   └── workflows/
│       └── deploy.yml          ← auto-deploy to GitHub Pages
├── public/
│   ├── quiz.html               ← اختبار المحاضرة (Telegram Mini App)
│   └── register.html           ← تسجيل الطلاب (Telegram Mini App)
├── n8n/
│   └── lecture-tracking-system.json  ← export كامل للـ workflow
└── README.md
```

---

## ⚙️ كيف يعمل النظام

### 1️⃣ تسجيل طالب جديد

```
الطالب يضغط /start في البوت
        ↓
البوت يبعتله زرار "تسجيل" يفتح register.html
        ↓
الطالب يدخل: الاسم + رقم الهاتف + يختار مجموعته
        ↓
POST /webhook/register-student
        ↓
n8n يحفظ البيانات في شيت Students
(appendOrUpdate بالـ chat_id كـ key — مينفعش طالب يتسجل مرتين)
```

**البيانات المحفوظة في Students sheet:**
| chat id | Student Name | Grade | Group Code | Phone Number | تاريخ الالتحاق |
|---------|-------------|-------|------------|-------------|----------------|

---

### 2️⃣ إرسال لنك المحاضرة تلقائياً

```
المدرس يكتب "post" في خلية (حصه 1 أو 2 ... إلخ) في شيت lectures
        ↓
Google Sheets Trigger (كل دقيقة يشوف التغييرات)
        ↓
n8n يقرأ: grop_id + lecture_num + url + title
        ↓
يجيب كل الطلاب اللي Group Code بتاعهم = grop_id
        ↓
يبني لكل طالب tracking URL:
https://alisaadeng-n8n.hf.space/webhook/lecture-track
  ?student_id=...&lecture_num=...&url=...
        ↓
يبعت لكل طالب رسالة Telegram تحتوي:
  📚 عنوان المحاضرة
  [▶️ مشاهدة المحاضرة] (tracking link)
        ↓
يحدث الخلية من "post" → "posted" (عشان متبعتش تاني)
```

---

### 3️⃣ تسجيل الحضور تلقائياً

```
الطالب يضغط على لنك المحاضرة
        ↓
GET /webhook/lecture-track?student_id=...&lecture_num=...&url=...
        ↓
n8n يتحقق: هل الـ User-Agent بوت؟
  ↓ لو بوت → Redirect مباشرة (بدون تسجيل)
  ↓ لو إنسان حقيقي:
        ↓
يجيب بيانات الطالب من Students sheet
        ↓
يكتب "حضر" في خلية (حصة N) في Students sheet
        ↓
Redirect للمحاضرة الأصلية (الطالب مش حاسس بأي حاجة)
```

> 💡 **عبقرية النظام:** الطالب بيضغط رابط واحد بس — بيتسجل حضوره تلقائياً وبعدين بينتقل للمحاضرة مباشرة.

---

### 4️⃣ إرسال بيانات الدرس للطلاب

```
Google Sheets Trigger (شيت Distribution — عمود stats)
        ↓
لما stats = "post":
  يجيب بيانات الدرس: Lessons + Learning Objectives + Vocabulary + Homework
        ↓
يجيب كل طلاب المجموعة
        ↓
يبعت لكل طالب رسالة Telegram تحتوي:
  📖 عنوان الدرس
  🎯 أهداف التعلم
  📝 المفردات
  🏠 الواجب
  [▶️ ابدأ الاختبار] → quiz Mini App مع lesson_code
        ↓
يحدث stats → "posted"
```

---

### 5️⃣ الاختبار وحفظ النتيجة

```
الطالب يفتح quiz Mini App من رابط الدرس
        ↓
quiz.html يجيب الأسئلة من Google Sheets (gviz API مباشرة)
        ↓
الطالب يجاوب الأسئلة
        ↓
POST /webhook/quiz-result
  { student_id, student_name, group_id, lesson_code, score, total_points, ... }
        ↓
n8n يستخرج رقم الحصة من lesson_code → lesson_N
يكتب الدرجة (score/total) في خلية lesson_N في شيت المجموعة
```

---

## 📊 Google Sheets

**Spreadsheet ID:** `1eel7cukc1Gs0krOkVnI5oDIfNy2uYpl0fIUD5RNwj5k`

### الأوراق (Sheets)

| الورقة | الـ GID | الغرض |
|--------|---------|--------|
| `Students` | 1560841313 | بيانات الطلاب + حضورهم + درجاتهم |
| `Distribution` | 1317367072 | جدول المحاضرات + stats |
| `Group Cods` | 2077656384 | قائمة المجموعات |
| `lectures` | 1393625686 | روابط المحاضرات لكل مجموعة |
| `[Group Sheets]` | — | ورقة لكل مجموعة: حضور + درجات |

### بنية شيت `lectures`
| grop_id | note | url | imgeg | حصه 1 | حصه 2 | ... | حصه 8 |
|---------|------|-----|-------|--------|--------|-----|--------|
| grp_6am | | https://... | | **post** | | | |

> ✍️ المدرس يكتب `post` في الخلية المناسبة → النظام يتحرك تلقائياً.

### بنية شيت `Distribution`
| Group Code | Lesson Code | Session number | Type | Lessons | Topic | Learning objectives 1 | Vocabulary | Homework | stats |
|------------|-------------|---------------|------|---------|-------|----------------------|------------|---------|-------|
| grp_6am | MATH_L1 | 1 | | Algebra | ... | ... | ... | | **post** |

> ✍️ المدرس يكتب `post` في عمود `stats` → يتبعت بيانات الدرس للطلاب.

### بنية شيت `Students`
| chat id | Student Name | Grade | Group Code | Phone Number | تاريخ الالتحاق | حصة 1 | حصة 2 | ... |
|---------|-------------|-------|------------|-------------|----------------|--------|--------|-----|
| 123456789 | أحمد محمد | مجموعة 6 صباحاً | grp_6am | 01012345678 | 2025-01-01 | حضر | حضر | |

---

## 🔧 n8n Workflow

**Workflow:** `Lecture Tracking System islam`  
**ID:** `8vIOgZFSwViIxISI`  
**الحالة:** ✅ Active  
**Timezone:** Africa/Cairo

### الـ 7 Triggers والـ Webhooks

| # | الاسم | النوع | الرابط | الغرض |
|---|-------|-------|--------|--------|
| 1 | `ارسال اللنك للطلاب` | Sheets Trigger | — | يراقب شيت lectures، يبعت لنك المحاضرة |
| 2 | `Google Sheets Trigger` | Sheets Trigger | — | يراقب شيت Distribution، يبعت بيانات الدرس |
| 3 | `GET Classes` | GET Webhook | `/webhook/get-classes` | يرجع قائمة المجموعات |
| 4 | `POST Register` | POST Webhook | `/webhook/register-student` | يحفظ بيانات الطالب |
| 5 | `Webhook: تسجيل الحضور1` | GET Webhook | `/webhook/lecture-track` | يسجل الحضور ويعمل redirect |
| 6 | `quiz-result` | POST Webhook | `/webhook/quiz-result` | يحفظ نتيجة الاختبار |
| 7 | `Webhook` | POST Webhook | `/webhook/17b123c9-...` | Telegram Bot webhook |

### Flow رسم تفصيلي

#### إرسال لنك المحاضرة
```
ارسال اللنك للطلاب (Sheets Trigger: lectures → حصه 1-8)
  └→ حدد الحصة المتغيرة (Code: يلاقي أول خلية = "post")
       └→ اقرأ الطلبة1 (Sheets: Students فلتر بـ Group Code)
            └→ ابني اللنكات1 (Code: يبني tracking URL لكل طالب)
                 └→ Remove Duplicates1
                      └→ ابعت للطلبة1 (HTTP: Telegram sendMessage)
                           └→ Code in JavaScript (يجهّز بيانات التحديث)
                                └→ حدّث الحصة = posted (Sheets: post → posted)
```

#### تسجيل الحضور
```
Webhook: تسجيل الحضور1 (GET /lecture-track?student_id=...&lecture_num=...&url=...)
  └→ If (User-Agent ≠ Bot)
       ├→ [بوت] Redirect للمحاضرة (مباشرة بدون تسجيل)
       └→ [إنسان] Get row(s) in sheet1 (يجيب بيانات الطالب)
                    └→ جهّز بيانات الحضور (Code: حصة N = "حضر")
                         └→ سجّل الحضور1 (Sheets: appendOrUpdate)
                              └→ Redirect للمحاضرة1
```

#### إرسال بيانات الدرس
```
Google Sheets Trigger (Distribution → stats = "post")
  └→ Limit
       └→ distribution sheet (يجيب بيانات الدرس)
            └→ student (يجيب طلاب المجموعة)
                 └→ to student2 (HTTP: Telegram رسالة شاملة بكل البيانات)
                      └→ Limit1
                           └→ Append or update row in sheet1 (stats → "posted")
```

#### حفظ نتيجة الاختبار
```
quiz-result (POST /quiz-result)
  └→ Code in JavaScript1
       يستخرج: lesson_code → lesson_N
       يجهّز: { 'chat id': ..., lesson_N: 'score/total' }
  └→ Append or update row in sheet2
       يكتب الدرجة في شيت المجموعة
```

---

## 🔗 Telegram Proxy

بدل Telegram API مباشرة، المشروع بيستخدم proxy على Vercel:

```
https://telegram-proxy-islam.vercel.app/api/telegram?method=sendMessage
```

ده بيحل مشكلة CORS ويخفي الـ Bot Token من الكود.

---

## 🚀 GitHub Actions

ملف `.github/workflows/deploy.yml` بيعمل deploy تلقائي على **GitHub Pages** عند كل push على `main`.

```yaml
name: Deploy to GitHub Pages
on:
  push:
    branches: [main]
  workflow_dispatch:

permissions:
  contents: read
  pages: write
  id-token: write

jobs:
  deploy:
    runs-on: ubuntu-latest
    environment:
      name: github-pages
      url: ${{ steps.deployment.outputs.page_url }}
    steps:
      - uses: actions/checkout@v4
      - uses: actions/configure-pages@v5
      - uses: actions/upload-pages-artifact@v3
        with:
          path: public/
      - uses: actions/deploy-pages@v4
        id: deployment
```

**الروابط بعد النشر:**
```
https://<username>.github.io/<repo>/quiz.html
https://<username>.github.io/<repo>/register.html
```

---

## 🗺️ الخارطة المستقبلية

- [ ] لوحة تحكم للمدرس لعرض إحصائيات الحضور والدرجات
- [ ] إشعار للمدرس لما الطالب يخلص الاختبار
- [ ] منع إعادة نفس الاختبار مرتين
- [ ] دعم الصور في أسئلة الاختبار
- [ ] تقرير أسبوعي تلقائي بالحضور

---

## 👨‍💻 المطور

**Ali saad glal automation specialist

---
<div align="center"><sub>Built with ❤️ for better math education</sub></div>
"# rageh-math-quiz" 
