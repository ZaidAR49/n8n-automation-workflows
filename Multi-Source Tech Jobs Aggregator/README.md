# مُجمِّع الوظائف التقنية متعددة المصادر (Multi-Source Tech Jobs Aggregator)

**Author:** Zaid Alradaideh
**Email:** zaidradaideh.dev@gmail.com

---

## نظرة عامة (Overview)

هذا الـ workflow مبني على n8n، ويقوم بتجميع الوظائف التقنية (Tech Jobs) من عدة مصادر مختلفة فور استقبال طلب عبر Webhook. يتم إرسال الطلب بالتوازي إلى أربعة مصادر: Remotive وRemoteOK وHacker News، بالإضافة إلى بوابة التوظيف الحكومية الأردنية (SPAC). بعد جلب البيانات الخام، يتم دمجها وتوحيد بنيتها (Normalize) في تنسيق موحد، ثم فلترتها للاحتفاظ فقط بالوظائف التقنية بالاعتماد على قاموس كلمات مفتاحية ثنائي اللغة (عربي/إنجليزي)، وأخيرًا يتم إرجاع النتائج النهائية كرد JSON على طلب الـ Webhook الأصلي مع دعم CORS.

---

## المصادر (Data Sources)

| المصدر (Source) | النوع (Type) |
|---|---|
| Remotive API | REST API |
| RemoteOK API | REST API (يتطلب User-Agent header) |
| Hacker News Job Stories | Firebase REST API |
| SPAC Jordan (بوابة التوظيف الحكومية الأردنية) | HTML Scraping |

---

## آلية العمل (Workflow Flow)

```
Webhook
  ├── HTTP Request - Remotive ─┐
  ├── HTTP Request - RemoteOK ─┤
  ├── HTTP Request - HackerNews┤
  └── jordan Government ──> Code in JavaScript ─┤
                                                 ▼
                                              Merge
                                                 ▼
                                     Normalize & Aggregate
                                                 ▼
                              Validate Tech Jobs (Binary Filter)
                                                 ▼
                                       Respond to Webhook
```

1. **Webhook** — نقطة الدخول الرئيسية، يستقبل الطلب ويستخدم `responseMode: responseNode`.
2. **HTTP Requests** — تُنفَّذ بالتوازي لجلب البيانات الخام من المصادر الأربعة.
3. **Code in JavaScript** — يحلّل HTML بوابة SPAC الأردنية ويستخرج الوظائف (title, company, url) مع إزالة التكرارات.
4. **Merge** — يدمج مخرجات المصادر الأربعة في مجرى بيانات واحد.
5. **Normalize & Aggregate** — يوحّد بنية البيانات القادمة من كل مصدر إلى شكل موحد: `title, company, url, source, tags`.
6. **Validate Tech Jobs (Binary Filter)** — يفلتر النتائج بالاعتماد على قاموس كلمات مفتاحية (عربي/إنجليزي) ليحتفظ فقط بالوظائف التقنية.
7. **Respond to Webhook** — يعيد جميع العناصر المفلترة كـ JSON، مع تفعيل CORS عبر `Access-Control-Allow-Origin: *`.

---

## بنية الاستجابة (Response Example)

```json
[
  {
    "title": "Frontend Developer",
    "company": "Example Co.",
    "url": "https://example.com/job/123",
    "source": "Remotive",
    "tags": ["react", "javascript"]
  }
]
```

---

## طريقة الاستخدام (Usage)

يتم تشغيل الـ workflow عبر إرسال طلب `GET` أو `POST` إلى رابط الـ Webhook:

```bash
curl https://<your-n8n-instance>/webhook/946430df-d514-426c-88c0-d4e232b17ca7
```

سيُعاد رد بصيغة JSON يحتوي على قائمة الوظائف التقنية المفلترة من جميع المصادر.

---

## المتطلبات (Requirements)

- نسخة فعالة من **n8n** (self-hosted أو cloud).
- اتصال بالإنترنت للوصول إلى الـ APIs الخارجية (Remotive, RemoteOK, Hacker News) وبوابة SPAC الأردنية.
- لا حاجة لأي **credentials** أو API keys — جميع المصادر عامة (public endpoints).

---

## ملاحظات (Notes)

- تم تفعيل **CORS** على مستوى الـ Respond to Webhook للسماح بالوصول من أي نطاق.
- فلترة الوظائف التقنية تعتمد على قاموس كلمات مفتاحية قابل للتوسعة (`TECH_KEYWORDS`) يدعم النصوص العربية والإنجليزية.
- عملية استخراج بيانات بوابة SPAC الأردنية تعتمد على HTML parsing عبر Regex، لذا أي تغيير في بنية الصفحة المصدر قد يتطلب تحديث الكود.
