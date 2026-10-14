# پلن توسعه — سامانه هوشمند اطلاع‌رسانی و پاسخگویی بحران (آگاه‌یار)

نسخه ۱.۱ · ۱۴ اکتبر ۲۰۲۶ · تغییر: Keycloak مرجع هویت و Audit · مبنا: [SRS](SRS.md) و [SAS](SAS.md)

این سند کار ۸ ماهه را به ۸ فاز تقسیم می‌کند. برای هر فاز، نیازمندی‌های SRS، خروجی و معیار اتمام آن آمده است. پرامپت‌های آماده هر بسته کاری هم در بخش ۸ است. خروجی ماه ۴ (MVP) پایان فاز ۳ و خروجی ماه ۸ (پایلوت) پایان فاز ۷ است.

---

## ۱. تصمیمات فنی پایه

| حوزه | انتخاب | توضیح |
| --- | --- | --- |
| زبان بک‌اند | Python 3.12 | |
| فریم‌ورک API | FastAPI + Uvicorn/Gunicorn | OpenAPI خودکار، async |
| مدل داده و اعتبارسنجی | Pydantic v2 + pydantic-settings | همه ورودی/خروجی API و رویدادهای Kafka با Pydantic |
| ORM و مهاجرت | SQLAlchemy 2 (async) + Alembic | |
| پایگاه داده | PostgreSQL 16 + PostGIS + pgvector | داده مکانی و برداری در یک موتور |
| صف رویداد | Apache Kafka (KRaft) + aiokafka | ستون Speed Layer |
| کش و قفل | Redis | Rate limit، session، کش |
| ذخیره شیء | MinIO | لایه Raw، Archive، فایل‌ها |
| هویت و Audit | Keycloak 26 (on-prem، نسخه pin‌شده) | OAuth2/OIDC، JWT، MFA، محدوده جغرافیایی با Group؛ مرجع Audit (بخش ۱-۳)؛ بدون وابستگی خارجی |
| مدل زبانی | LLM محلی (مثل GPT-OSS) روی vLLM با API سازگار OpenAI | هیچ فراخوانی به سرویس خارج از کشور |
| NLP فارسی | Hazm، ParsBERT، spaCy | |
| رابط کاربری | React 18 + TypeScript (strict) + Vite | |
| کیت UI | MUI v5 با RTL (stylis-plugin-rtl) | پشتیبانی کامل راست‌به‌چپ |
| داده سمت کلاینت | TanStack Query + کلاینت تولیدشده از OpenAPI (openapi-typescript) | |
| فرم | react-hook-form + zod | |
| نقشه | MapLibre GL با کاشی‌های self-hosted | بدون CDN و کاشی خارجی |
| تاریخ | dayjs + jalaliday | تقویم شمسی |
| فونت | Vazirmatn (self-hosted) | |
| مشاهده‌پذیری | Prometheus، Grafana، Loki، OpenTelemetry | همه self-hosted |
| استقرار | Docker، Kubernetes + Helm | |

### ۱-۱. انحراف آگاهانه از SAS: ماژولار مونولیت در فاز اول

SAS چهارده سرویس تعریف کرده است. تیم چهار نفره است (جدول ۸ پروپوزال)؛ اجرای ۱۴ سرویس از روز اول هزینه عملیاتی بالایی دارد. پیشنهاد:

- **تا ماه ۴:** یک برنامه FastAPI با ماژول‌های جدا، به‌ازای هر سرویس SAS یک ماژول. مرز ماژول‌ها سخت است: هیچ ماژولی به جدول ماژول دیگر دسترسی مستقیم ندارد. ارتباط فقط از طریق `service` عمومی یا رویداد Kafka است.
- **از ماه ۵:** سه بخش پرفشار جدا مستقر می‌شوند: `stream-worker` (T3)، `dispatcher` (T8) و `nlp-service` (T6، روی GPU).
- بقیه ماژول‌ها هر وقت بار واقعی لازم کرد جدا می‌شوند. چون مرزها از اول رعایت شده، جداسازی بدون بازنویسی ممکن است.

این تصمیم باید در ADR-08 ثبت و به تأیید کارفرما برسد.

### ۱-۲. اپلیکیشن موبایل

جدول ۳ پروپوزال اپلیکیشن موبایل شهروندی را خواسته است. برای پایلوت، **PWA** (همان React) پیشنهاد می‌شود: عکس، موقعیت و نوتیفیکیشن را پوشش می‌دهد و نیاز به انتشار در فروشگاه ندارد. اپ بومی به بعد از پایلوت موکول می‌شود. این مورد نیز نیازمند تأیید کارفرماست.

### ۱-۳. Keycloak به‌عنوان مرجع هویت و Audit

Keycloak هم هویت و دسترسی را مدیریت می‌کند و هم مرجع Audit سامانه است. پیکربندی آن کاملاً با اسکریپت انجام می‌شود (پرامپت P0.5) و هیچ تنظیمی دستی در کنسول انجام نمی‌شود.

**ریلم‌ها:**

| ریلم | کاربران | کلاینت‌ها | ورود |
| --- | --- | --- | --- |
| `master` | فقط ادمین Keycloak | — | کنسول ادمین فقط از شبکه مدیریت؛ MFA |
| `crisis-ops` | مدیران ملی، استانی، محلی، کارشناسان، کارشناس داده، مدیر سامانه | `crisis-web` (public، PKCE)، `crisis-api` (confidential، bearer-only)، `crisis-worker` (service account برای سرویس‌ها)، `crisis-audit-reader` (service account فقط‌خواندنی رویدادها) | رمز + OTP اجباری برای نقش‌های مدیر و تأییدکننده |
| `crisis-citizen` | شهروندان | `citizen-pwa` (public، PKCE) | ثبت‌نام آزاد؛ ورود فقط برای ارسال گزارش؛ مشاهده هشدار بدون ورود |

**محدوده جغرافیایی:** ساختار Group سلسله‌مراتبی در `crisis-ops`: `/national`، `/province/tehran`، `/province/tehran/city/<name>` و همین الگو برای مرکزی و هرمزگان. هر Group ویژگی `geo_scope` دارد و یک Protocol Mapper آن را به همراه نقش‌ها در توکن قرار می‌دهد. بک‌اند محدوده را فقط از توکن می‌خواند.

**Audit در Keycloak:**

- **رویدادهای کاربر** (ورود، خروج، ورود ناموفق، OTP، تمدید توکن، تغییر رمز) و **رویدادهای ادمین** (ایجاد و تغییر کاربر، نقش، Group، کلاینت، با جزئیات) در هر سه ریلم فعال است، با زمان نگهداری مصوب.
- **اقدامات کسب‌وکاری** (ثبت بحران، تأیید و رد پیام، انتشار، توقف اضطراری، تغییر قاعده، آزادسازی قرنطینه) هم به Keycloak فرستاده می‌شوند. Keycloak برای ثبت رویداد از بیرون API عمومی ندارد؛ بنابراین یک افزونه SPI جاوا به نام `crisis-audit-spi` ساخته می‌شود که:
    1. یک Realm Resource با مسیر `/realms/crisis-ops/crisis-audit/events` اضافه می‌کند که فقط کلاینت `crisis-api` با نقش `audit-writer` می‌تواند رویداد بنویسد. رویداد شامل شناسه کاربر، شناسه session کیکلاک، اقدام، منبع، محدوده جغرافیایی و نتیجه است.
    2. یک Event Listener که همه رویدادهای کاربر، ادمین و کسب‌وکاری را با زنجیره هش در جدول append-only ذخیره و همزمان به SIEM/SOC کارفرما (syslog یا CEF) ارسال می‌کند.
- **مشاهده و گزارش:** صفحه Audit در پنل مدیریت از طریق کلاینت `crisis-audit-reader` رویدادها را از Keycloak می‌خواند؛ برنامه هیچ جدول Audit جداگانه‌ای ندارد.
- **تاب‌آوری:** اگر Keycloak در دسترس نباشد، بک‌اند رویداد کسب‌وکاری را در Outbox محلی نگه می‌دارد و پس از بازگشت ارسال می‌کند؛ هیچ اقدامی بدون ثبت Audit از دست نمی‌رود.

**ریسک این انتخاب:** Keycloak برای Audit کسب‌وکاری طراحی نشده است. نگهداری افزونه SPI با هر ارتقای نسخه Keycloak باید آزمون شود و حجم رویداد کسب‌وکاری بار پایگاه داده Keycloak را بالا می‌برد. این تصمیم در ADR-09 ثبت می‌شود.

---

## ۲. استانداردهای کدنویسی

### ۲-۱. بک‌اند (PEP 8 و استانداردهای مرتبط)

| استاندارد | ابزار اعمال | قاعده |
| --- | --- | --- |
| PEP 8 (سبک کد) | `ruff check` (قواعد E، W، N) + `ruff format` | طول خط ۷۹؛ نام‌گذاری `snake_case` برای تابع و متغیر، `PascalCase` برای کلاس، `UPPER_CASE` برای ثابت |
| PEP 257 (Docstring) | `ruff` (قواعد D) | Docstring برای همه ماژول‌ها، کلاس‌ها و توابع عمومی |
| PEP 484/604 (Type hints) | `mypy --strict` | بدون `Any` بی‌دلیل؛ همه توابع تایپ‌دار |
| ترتیب import | `ruff` (قاعده I) | stdlib، third-party، local |
| امنیت کد | `ruff` (قاعده S = bandit) + Semgrep | |
| تست | `pytest`، `pytest-asyncio`، `httpx`، testcontainers | پوشش ≥ ۸۰٪ برای `service` و `rules` |

پیکربندی نمونه `pyproject.toml`:

```toml
[tool.ruff]
line-length = 79
target-version = "py312"

[tool.ruff.lint]
select = ["E", "W", "F", "N", "I", "D", "UP", "B", "S", "ASYNC", "C4", "SIM", "RUF"]
ignore = ["D203", "D213"]

[tool.ruff.lint.pydocstyle]
convention = "pep257"

[tool.mypy]
strict = true
plugins = ["pydantic.mypy"]
```

قواعد Pydantic:

- همه بدنه‌های درخواست و پاسخ مدل Pydantic دارند؛ هیچ endpoint ای `dict` خام برنمی‌گرداند.
- `model_config = ConfigDict(extra="forbid")` برای ورودی‌ها (جلوگیری از Mass Assignment).
- مدل‌های ورودی (`*Create`، `*Update`) از مدل خروجی (`*Read`) جدا هستند.
- محدودیت طول و الگو با `Field(max_length=..., pattern=...)` روی همه رشته‌ها.
- رویدادهای Kafka نیز مدل Pydantic نسخه‌دار دارند (`schema_version`).

### ۲-۲. فرانت‌اند

- TypeScript `strict`، ESLint (typescript-eslint، react-hooks، jsx-a11y) و Prettier.
- هیچ اسکریپت، فونت، کاشی نقشه یا CDN خارجی؛ همه دارایی‌ها داخل build.
- رشته‌های رابط کاربری از فایل i18n؛ پیش‌فرض فارسی و RTL.
- تست: Vitest + React Testing Library؛ سناریوهای کامل با Playwright.

### ۲-۳. ساختار مخزن

```text
crisis/
├── backend/
│   ├── pyproject.toml
│   ├── alembic/
│   └── app/
│       ├── main.py
│       ├── core/            # config، logging، db، security، kafka، errors
│       ├── modules/
│       │   ├── iam/         # T11 — اعتبارسنجی توکن Keycloak، مجوز، محدوده جغرافیایی
│       │   ├── audit/       # T11 — کلاینت Audit کیکلاک (نوشتن با Outbox، خواندن)
│       │   ├── crisis/      # T1، T10 — پرونده بحران، State Machine
│       │   ├── sources/     # T2 — Source Registry، Connectorها
│       │   ├── stream/      # T3 — پردازش بلادرنگ
│       │   ├── quality/     # T4 — ETL، کیفیت، قرنطینه، Lineage
│       │   ├── rules/       # T5 — Rule Engine
│       │   ├── ml/          # T5 — امتیاز ریسک، Nowcasting
│       │   ├── nlp/         # T6 — NER، Intent، RAG، Guardrail
│       │   ├── workflow/    # T7 — صف کار، تأیید، SLA
│       │   ├── dispatch/    # T8 — انتشار چندکاناله
│       │   ├── dashboard/   # T9 — KPI، GIS، گزارش
│       │   └── citizen/     # T9 — پورتال شهروندی، گزارش مردمی
│       └── workers/         # نقطه ورود مصرف‌کننده‌های Kafka
├── frontend/
│   └── src/
│       ├── app/             # routing، layout، auth
│       ├── features/        # هم‌نام با ماژول‌های بک‌اند
│       ├── shared/          # کامپوننت‌ها، api client، i18n
│       └── citizen/         # PWA شهروندی
├── infra/
│   ├── docker-compose.yml
│   ├── helm/
│   └── keycloak/
│       ├── bootstrap.sh         # اسکریپت idempotent ساخت ریلم‌ها با kcadm.sh
│       ├── config/              # تعریف ریلم، نقش، Group و کلاینت (منبع حقیقت)
│       ├── crisis-audit-spi/    # افزونه جاوا: API ثبت رویداد + Event Listener
│       └── themes/crisis-fa/    # تم فارسی و RTL صفحه ورود
└── docs/
```

ساختار داخلی هر ماژول بک‌اند:

```text
modules/<name>/
├── models.py       # SQLAlchemy
├── schemas.py      # Pydantic
├── repository.py   # دسترسی به داده
├── service.py      # منطق کسب‌وکار (تنها رابط عمومی ماژول)
├── router.py       # FastAPI APIRouter
├── events.py       # رویدادهای Kafka (Pydantic)
└── tests/
```

---

## ۳. الزامات افتا

سامانه مدیریت بحران در دسته زیرساخت حیاتی است و پیش از بهره‌برداری باید ارزیابی امنیتی افتا (مرکز مدیریت راهبردی افتا) را بگذراند. کنترل‌های زیر از روز اول در طراحی و کد اعمال می‌شوند تا ارزیابی پایان کار به بازنویسی منجر نشود.

> **مورد باز:** فهرست دقیق الزامات، نسخه چک‌لیست و سطح ارزیابی مورد نیاز (از جمله اینکه ارزیابی محصول بر پایه معیار مشترک ISIRI-ISO/IEC 15408 لازم است یا خیر، و الگوریتم‌های رمز مورد قبول) باید از کارفرما یا آزمایشگاه مورد تأیید افتا گرفته شود. کنترل‌های این بخش بر پایه ISO/IEC 27001، OWASP ASVS سطح ۲ و الزامات رایج سامانه‌های زیرساختی تنظیم شده و پس از دریافت چک‌لیست رسمی تطبیق داده می‌شود.

| شناسه | حوزه | کنترل | فاز | پیاده‌سازی |
| --- | --- | --- | --- | --- |
| AFTA-01 | بومی‌سازی و حاکمیت داده | همه داده، لاگ و مدل در مرکز داده داخل کشور؛ هیچ فراخوانی به سرویس خارجی (LLM، نقشه، CDN، فونت، تلمتری) | ۰ | سیاست egress شبکه deny-by-default؛ تست خودکار عدم وجود URL خارجی در build |
| AFTA-02 | زنجیره تأمین نرم‌افزار | همه وابستگی‌ها از مخزن آینه داخلی (Nexus/Artifactory)؛ نسخه‌ها pin؛ SBOM برای هر انتشار | ۰ | `uv lock`، `package-lock.json`، CycloneDX |
| AFTA-03 | احراز هویت | OIDC با Keycloak؛ MFA اجباری برای نقش‌های مدیر و تأییدکننده؛ سیاست رمز عبور و Brute Force Detection | ۰ | `infra/keycloak/bootstrap.sh` (P0.5) |
| AFTA-04 | کنترل دسترسی | RBAC سه‌سطحی (نقش‌های ریلم) + محدوده جغرافیایی (Group کیکلاک)؛ بررسی مجوز در هر endpoint سمت سرور | ۱ | وابستگی FastAPI `require_permission` |
| AFTA-05 | تفکیک وظایف | نویسنده پیام نمی‌تواند تأییدکننده همان پیام باشد؛ تغییر قاعده نیاز به تأیید نفر دوم دارد | ۲ | قاعده در Workflow |
| AFTA-06 | مدیریت نشست | توکن دسترسی کوتاه‌عمر (≤ ۱۵ دقیقه)، refresh چرخشی، خروج سراسری، timeout بی‌فعالیتی | ۰ | تنظیمات ریلم در P0.5 + فرانت |
| AFTA-07 | رویدادنگاری و حسابرسی | Keycloak مرجع واحد Audit: رویدادهای کاربر و ادمین + اقدامات کسب‌وکاری از طریق `crisis-audit-spi`؛ زنجیره هش، append-only، ارسال به SIEM/SOC کارفرما (syslog/CEF) | ۰–۱ | P0.5، P1.1 |
| AFTA-08 | همگام‌سازی زمان | همه سرورها با NTP داخلی؛ زمان رویدادها UTC با منطقه زمانی صریح | ۰ | زیرساخت |
| AFTA-09 | رمزنگاری در انتقال | TLS 1.2+ (ترجیحاً 1.3) برای همه ارتباطات، از جمله داخل کلاستر (mTLS) | ۶ | Ingress + service mesh یا mTLS |
| AFTA-10 | رمزنگاری در ذخیره | رمزنگاری دیسک و پشتیبان؛ رمزنگاری سطح فیلد برای داده شخصی شهروند (شماره تلفن، موقعیت دقیق) | ۲ | `cryptography` + KMS/Vault داخلی |
| AFTA-11 | مدیریت اسرار | هیچ رمز یا کلید در کد و image؛ Vault یا Kubernetes Secrets رمزنگاری‌شده | ۰ | gitleaks در CI |
| AFTA-12 | اعتبارسنجی ورودی | Pydantic با `extra="forbid"` و محدودیت طول؛ پاک‌سازی فایل آپلودی (نوع، اندازه، اسکن بدافزار) | ۱ | |
| AFTA-13 | خطاها | هیچ stack trace یا جزئیات داخلی در پاسخ؛ پیام عمومی + شناسه پیگیری | ۰ | exception handler سراسری |
| AFTA-14 | سرآیندهای امنیتی | CSP سخت‌گیرانه، HSTS، X-Content-Type-Options، Referrer-Policy، Frame-Options | ۰ | middleware + Nginx |
| AFTA-15 | محدودیت نرخ و DoS | Rate limit به‌ازای کاربر و IP؛ محدودیت اندازه بدنه؛ timeout | ۳ | Redis + Nginx |
| AFTA-16 | امنیت هوش مصنوعی | Guardrail در برابر Prompt Injection؛ مدل فقط به اسناد تأییدشده دسترسی دارد؛ ثبت کامل ورودی و خروجی مدل | ۳ | ماژول `nlp` |
| AFTA-17 | سخت‌سازی | image حداقلی و non-root؛ CIS Benchmark برای OS، Kubernetes و PostgreSQL | ۶ | |
| AFTA-18 | تفکیک شبکه | جداسازی ناحیه‌ها: DMZ (پورتال شهروند)، ناحیه کاربری، ناحیه داده؛ NetworkPolicy | ۶ | |
| AFTA-19 | پشتیبان و بازیابی | پشتیبان رمزنگاری‌شده روزانه، تست بازیابی ماهانه، RPO و RTO مصوب | ۶ | |
| AFTA-20 | مدیریت آسیب‌پذیری | SAST (ruff-S، Semgrep)، SCA (pip-audit، npm audit)، اسکن image (Trivy)، DAST (OWASP ZAP) در CI؛ هیچ یافته High/Critical باز در انتشار | ۰ | |
| AFTA-21 | ارزیابی نفوذ | Pen-Test توسط آزمایشگاه مورد تأیید افتا پیش از پایلوت؛ رفع یافته‌ها | ۶ | |
| AFTA-22 | مستندات امنیتی | سند معماری امنیت، مدل تهدید (STRIDE)، چک‌لیست ASVS، رویه پاسخ به حادثه، Runbook | ۶ | `docs/security/` |
| AFTA-24 | امنیت Keycloak | کنسول ادمین و ریلم `master` فقط از شبکه مدیریت؛ پایگاه داده جدا؛ اجرای production mode با TLS؛ پشتیبان و تست بازیابی ریلم‌ها | ۶ | P6.3، P6.5 |
| AFTA-23 | مدیریت تغییر | هر تغییر از طریق Pull Request با بازبینی؛ شاخه main محافظت‌شده؛ امضای commit و image | ۰ | |

---

## ۴. نمای کلی فازها

| فاز | بازه | هدف | بسته‌های اصلی | خروجی |
| --- | --- | --- | --- | --- |
| ۰ | هفته ۱–۲ | زیرساخت توسعه، CI امن، اسکلت برنامه | T1، T11، T12 | مخزن آماده، CI سبز، محیط توسعه با یک فرمان |
| ۱ | هفته ۳–۸ | هویت، Audit، پرونده بحران، منابع داده، ستون Kafka | T1، T2، T3، T11 | تریگر دستی و ثبت منبع |
| ۲ | هفته ۹–۱۲ | ETL و کیفیت، Rule Engine، Workflow پایه، قالب‌ها | T4، T5، T7 | ارزیابی Rule-based و صف کارشناس |
| ۳ | هفته ۱۳–۱۷ | RAG پایه، انتشار وب و SMS، داشبورد، چرخه عمر پایه | T6، T8، T9، T10 | **MVP ماه ۴** |
| ۴ | ماه ۵ | پردازش بلادرنگ، کیفیت پیشرفته، هم‌بستگی، امتیاز ریسک | T2، T3، T4، T5 | Speed Layer عملیاتی |
| ۵ | ماه ۶ | چت‌بات، سه سطح تأیید عملیاتی، کانال‌ها، پورتال شهروندی | T5، T6، T7، T8، T9 | انتشار کنترل‌شده چندکاناله |
| ۶ | ماه ۷ | داشبورد عملیاتی، چرخه عمر کامل، پایش ۷×۲۴، بار، آمادگی افتا | T9، T10، T11، T12 | آمادگی UAT و ارزیابی افتا |
| ۷ | ماه ۸ | استقرار پایلوت در سه استان، UAT، آموزش | T12 | **پایلوت ماه ۸** |

---

## ۵. شرح فازها

### فاز ۰ — زیرساخت و اسکلت (هفته ۱–۲)

- **پوشش:** NFR-08، NFR-09، NFR-16، AFTA-01، 02، 03، 06، 07، 08، 11، 13، 14، 20، 23.
- **خروجی:** مخزن monorepo، `docker compose up` کل محیط را بالا می‌آورد، CI با lint، type check، تست، SAST، SCA و SBOM، Keycloak با ریلم‌های `crisis-ops` و `crisis-citizen` که فقط با اسکریپت ساخته می‌شوند.
- **معیار اتمام:** CI روی main سبز؛ `/health` و `/ready` پاسخ می‌دهند؛ اجرای دوباره `bootstrap.sh` بدون خطا و بدون تغییر است؛ ورود فرانت با Keycloak و تم فارسی کار می‌کند؛ ورود ناموفق در رویدادهای Keycloak دیده می‌شود.
- **پرامپت‌ها:** P0.1 تا P0.5 (P0.5 پیش از P0.4 اجرا شود).

### فاز ۱ — هویت، پرونده بحران، منابع (هفته ۳–۸)

- **پوشش:** FR-01.1، FR-02.1، FR-05.1، FR-17.1 (اسکلت)، TR-T1، TR-T2.1، TR-T11، NFR-12، AFTA-04، 05، 07، 12.
- **خروجی:** مجوزدهی با نقش و Group کیکلاک، ثبت اقدامات کسب‌وکاری در Audit کیکلاک، ثبت دستی بحران و تحلیل اولیه، Source Registry با کانکتورهای فایل، REST، DB و RSS، رویدادهای Kafka با Outbox.
- **معیار اتمام:** سناریوی «کارشناس استان تهران بحران زلزله را ثبت می‌کند» end-to-end اجرا شود و رویداد آن در Audit کیکلاک دیده شود؛ کاربر استان مرکزی پرونده تهران را نبیند.
- **پرامپت‌ها:** P1.1 تا P1.5.

### فاز ۲ — کیفیت، قواعد، گردش کار (هفته ۹–۱۲)

- **پوشش:** FR-03.1، FR-04.1، FR-06.1، FR-07.1، FR-08.1، FR-09.1، FR-10.1، FR-12.1، FR-13.1، TR-T4، TR-T5.1، TR-T7.1، AFTA-05، 10.
- **خروجی:** ETL و اعتبارسنجی Schema، لایه‌های Raw و Curated، Rule Engine نسخه‌دار، صف کار کارشناس، تأیید و رد پیش‌نویس، قالب‌های دستورالعمل.
- **معیار اتمام:** داده زلزله آزمایشی از ورود تا ارزیابی «قابل‌توجه» و ارجاع به کارشناس برسد؛ دلایل ارزیابی نمایش داده شود.
- **پرامپت‌ها:** P2.1 تا P2.4.

### فاز ۳ — MVP ماه ۴ (هفته ۱۳–۱۷)

- **پوشش:** FR-11.1، FR-14.1، FR-15.1، FR-16.1، FR-17.1، FR-18.1، FR-19.1، TR-T6.2، TR-T8 (پایه)، AFTA-15، 16.
- **خروجی:** پیش‌نویس پیام با RAG و نمایش منبع، انتشار Web/API و SMS آزمایشی، داشبورد فنی و نقشه ساده، تاریخچه، بستن دستی، بایگانی، پایش زمان‌بندی‌شده.
- **معیار اتمام:** همه نیازمندی‌های «.1» بخش ۳ SRS در محیط دمو با سناریوی زلزله اجرا شود (بخش ۷-۱ SRS). تست Playwright سناریوی کامل سبز باشد.
- **پرامپت‌ها:** P3.1 تا P3.4.

### فاز ۴ — هوشمندی و بلادرنگ (ماه ۵)

- **پوشش:** FR-01.2، FR-05.2، FR-06.2، FR-07.2، FR-08.2، TR-T2.2، TR-T3، TR-T4.1، TR-T4.2، NFR-01، NFR-07.
- **خروجی:** Stream worker مستقل با Window، Dedup و Checkpoint؛ تشخیص خودکار با هم‌بستگی چند منبع؛ NER و امتیاز کیفیت و قرنطینه؛ DLQ با رابط بازبینی؛ امتیاز ریسک و مدل پایه Nowcasting.
- **معیار اتمام:** تأخیر p95 از ورود رویداد تا هشدار اولیه کمتر از ۵۰۰ms در تست بار؛ قطع یک منبع بدون از دست رفتن داده.
- **پرامپت‌ها:** P4.1 تا P4.4.

### فاز ۵ — تعامل و انتشار کنترل‌شده (ماه ۶)

- **پوشش:** FR-03.2، FR-04.2، FR-09.2، FR-10.2، FR-11.2، FR-12.2، FR-13.2، FR-14.2، FR-20 تا FR-23، TR-T6.1، TR-T7.1، TR-T8.1، NFR-02، NFR-13.
- **خروجی:** چت‌بات شهروندی، سه سطح تأیید عملیاتی با SLA و Escalation، دکمه توقف اضطراری، آداپتورهای SMS، Push و پیام‌رسان‌ها، گردش کار خبر مشکوک، PWA شهروندی.
- **معیار اتمام:** Intent ≥ ۹۰٪ و F1 جستجو ≥ ۰٫۸۵ روی مجموعه آزمون؛ پیام سطح ۳ بدون زنجیره تأیید به هیچ مسیری منتشر نشود (تست منفی)؛ تحویل SMS/Push ≥ ۹۵٪ در محیط تست.
- **پرامپت‌ها:** P5.1 تا P5.5.

### فاز ۶ — عملیات و آمادگی امنیتی (ماه ۷)

- **پوشش:** FR-15.2، FR-16.2، FR-17.2، FR-18.2، FR-19.2، TR-T9.1، TR-T10، NFR-03 تا NFR-07، NFR-10، NFR-14، NFR-17، AFTA-09، 17 تا 22.
- **خروجی:** داشبورد عملیاتی سلسله‌مراتبی و خروجی PDF/Excel، به‌روزرسانی خودکار و پیشنهاد خاتمه، سیاست نگهداری، مشاهده‌پذیری کامل، تست بار و Chaos، بسته مستندات افتا، Pen-Test.
- **معیار اتمام:** تست بار ۱۰۰۰ کاربر همزمان موفق؛ تمرین Fail/Recover مستند؛ هیچ یافته High/Critical باز در Pen-Test.
- **پرامپت‌ها:** P6.1 تا P6.5.

### فاز ۷ — پایلوت ماه ۸

- **پوشش:** بخش ۷-۱ SRS (معیار پایان ماه ۸)، NFR-15، NFR-17.
- **خروجی:** استقرار در تهران، مرکزی و هرمزگان، UAT، Runbook، آموزش، گزارش پذیرش هر استان.
- **معیار اتمام:** همه نیازمندی‌های «.2» در سه استان اجرا و پذیرش UAT امضا شود.
- **پرامپت‌ها:** P7.1.

---

## ۶. نگاشت فاز به نیازمندی

| نیازمندی | M4 (فاز) | M8 (فاز) |
| --- | --- | --- |
| FR-01 تریگر | ۱ | ۴ |
| FR-02 تحلیل اولیه | ۱ | ۴ |
| FR-03 دستورالعمل کلی | ۲ | ۵ |
| FR-04 شاخص ارزیابی | ۲ | ۵ |
| FR-05 جمع‌آوری | ۱ | ۴ |
| FR-06 پیش‌پردازش | ۲ | ۴ |
| FR-07 ابعاد و اثرات | ۲ | ۴ |
| FR-08 ارزیابی نهایی | ۲ | ۴ |
| FR-09 ارجاع به کارشناس | ۲ | ۵ |
| FR-10 تصمیم انتشار | ۲ | ۵ |
| FR-11 سطح ۱ | ۳ | ۵ |
| FR-12 سطح ۲ | ۲ | ۵ |
| FR-13 سطح ۳ | ۲ | ۵ |
| FR-14 انتشار | ۳ | ۵ |
| FR-15 داشبورد | ۳ | ۶ |
| FR-16 به‌روزرسانی | ۳ | ۶ |
| FR-17 بستن/تداوم | ۳ | ۶ |
| FR-18 بایگانی | ۳ | ۶ |
| FR-19 پایش مداوم | ۳ | ۶ |
| FR-20 تا FR-23 | — | ۵ |

---

## ۷. روش استفاده از پرامپت‌ها

1. پرامپت‌ها به ترتیب اجرا می‌شوند؛ هر پرامپت روی یک شاخه جدا و با یک Pull Request.
2. پرامپت P0.1 فایل `CLAUDE.md` را در ریشه مخزن می‌سازد. این فایل قواعد ثابت پروژه را دارد و در هر جلسه خوانده می‌شود؛ پرامپت‌های بعدی آن را تکرار نمی‌کنند.
3. P0.5 پیش از P0.4 اجرا می‌شود چون فرانت به ریلم آماده نیاز دارد. هر پرامپت شناسه نیازمندی‌های SRS را نام می‌برد. پیش از شروع، بخش مربوط در `docs/SRS.md` و `docs/SAS.md` خوانده شود.
4. PR فقط وقتی ادغام می‌شود که CI سبز باشد و معیارهای پذیرش پرامپت تیک خورده باشد.
5. هر تصمیم معماری جدید به‌صورت ADR در `docs/adr/` ثبت می‌شود.

---

## ۸. پرامپت‌های توسعه

### فاز ۰

#### P0.1 — اسکلت monorepo و ابزارها

```text
مستندات docs/SRS.md و docs/SAS.md و docs/DEVELOPMENT_PLAN.md (بخش‌های ۱ تا ۳) را بخوان.
اسکلت monorepo را طبق بخش ۲-۳ پلن بساز:
- backend/: پروژه Python 3.12 با uv، FastAPI، Pydantic v2، pydantic-settings،
  SQLAlchemy 2 async، Alembic، pytest. پیکربندی ruff و mypy دقیقاً طبق بخش ۲-۱
  (line-length=79، convention=pep257، mypy strict).
- frontend/: Vite + React 18 + TypeScript strict، ESLint، Prettier، Vitest.
- infra/docker-compose.yml: PostgreSQL 16 با PostGIS و pgvector، Kafka در حالت KRaft،
  Redis، MinIO، Keycloak 26 با پایگاه داده PostgreSQL جداگانه (نه H2، نه DB برنامه).
  همه image ها با نسخه pin شده؛ هیچ پورتی جز موارد لازم باز نباشد.
  سرویس keycloak-bootstrap که پس از سالم شدن Keycloak، infra/keycloak/bootstrap.sh
  را اجرا کند (محتوای اسکریپت در P0.5 ساخته می‌شود؛ اینجا فقط جای آن را بگذار).
- pre-commit با ruff، ruff-format، mypy، eslint، gitleaks.
- Makefile با فرمان‌های dev، test، lint، typecheck، migrate.
- CLAUDE.md در ریشه با خلاصه قواعد بخش ۲ و ۳ پلن.
معیار پذیرش:
- `make dev` کل محیط را بالا می‌آورد.
- `make lint typecheck test` بدون خطا اجرا می‌شود.
- هیچ URL خارجی (CDN، فونت، API) در کد فرانت وجود ندارد (AFTA-01).
```

#### P0.2 — CI امن

```text
یک pipeline در .github/workflows بساز (طوری بنویس که انتقال به GitLab CI داخلی ساده باشد):
- backend: ruff check، ruff format --check، mypy، pytest با پوشش.
- frontend: eslint، tsc --noEmit، vitest، build.
- امنیت (AFTA-02، AFTA-11، AFTA-20): gitleaks، Semgrep (قواعد python و typescript)،
  pip-audit، npm audit --audit-level=high، Trivy روی image ها،
  تولید SBOM با CycloneDX برای backend و frontend به‌عنوان artifact.
- اسکریپت scripts/check_no_external_urls.py که build فرانت و کد بک‌اند را برای
  دامنه‌های خارجی (به‌جز موارد مجاز در فایل allowlist) بررسی کند و در CI اجرا شود.
- هر یافته High یا Critical باعث شکست pipeline شود.
معیار پذیرش: pipeline روی main سبز است؛ یک commit آزمایشی حاوی secret جعلی شکست می‌خورد.
```

#### P0.3 — هسته بک‌اند

```text
در backend/app/core پیاده کن:
- config.py: Settings با pydantic-settings؛ همه اسرار فقط از متغیر محیطی؛
  اعتبارسنجی هنگام راه‌اندازی.
- logging.py: لاگ JSON ساخت‌یافته با request_id و user_id؛ زمان UTC؛
  هیچ داده شخصی در لاگ (ماسک شماره تلفن و توکن).
- db.py: AsyncEngine، AsyncSession، Base؛ اولین migration با extension های
  postgis و vector.
- errors.py: exception handler سراسری؛ پاسخ خطا مدل Pydantic ErrorResponse با
  code، message و trace_id؛ هیچ stack trace به کاربر نرسد (AFTA-13).
- security_headers.py: middleware برای CSP، HSTS، X-Content-Type-Options،
  Referrer-Policy، X-Frame-Options (AFTA-14).
- endpoints /health و /ready (بررسی DB، Kafka، Redis).
- محدودیت اندازه بدنه درخواست و timeout.
همه کد PEP 8، با docstring PEP 257 و تایپ کامل. برای هر بخش تست pytest بنویس.
معیار پذیرش: تست‌ها سبز؛ پاسخ خطای ۵۰۰ فقط trace_id دارد؛ سرآیندهای امنیتی در هر پاسخ هست.
```

#### P0.4 — پوسته فرانت‌اند

```text
در frontend:
- MUI v5 با RTL (stylis-plugin-rtl و @emotion/cache)، تم فارسی، فونت Vazirmatn
  به‌صورت self-hosted در public/fonts.
- i18n با react-i18next؛ زبان پیش‌فرض fa، فایل locales/fa.json.
- پیش‌نیاز: P0.5 اجرا شده باشد.
- ورود با OIDC Authorization Code + PKCE به ریلم crisis-ops و کلاینت crisis-web
  (oidc-client-ts)؛ آدرس Keycloak و نام ریلم از متغیر محیطی build؛
  توکن فقط در حافظه، نه localStorage؛ refresh خودکار؛ خروج سراسری (AFTA-06).
- نقش‌ها و geo_scope فقط برای نمایش منو از توکن خوانده شوند؛ تصمیم دسترسی همیشه سمت سرور است.
- layout با منوی کناری، نوار بالا با نام کاربر، نقش و استان (از geo_scope).
- react-router با مسیرهای محافظت‌شده بر اساس نقش.
- تولید خودکار کلاینت تایپ‌دار از OpenAPI بک‌اند با openapi-typescript و
  یک wrapper برای TanStack Query.
- dayjs با jalaliday برای نمایش تاریخ شمسی.
معیار پذیرش: کاربر آزمایشی وارد می‌شود و منوی متناسب با نقشش را می‌بیند؛ صفحه کاملاً RTL است؛
تست Vitest برای guard مسیرها.
```

#### P0.5 — راه‌اندازی Keycloak و ساخت ریلم‌ها با اسکریپت

```text
پلن: بخش ۱-۳ (ریلم‌ها، Group ها، Audit)، AFTA-03، AFTA-06، AFTA-07. SRS: NFR-09، NFR-12.
هیچ تنظیمی در کنسول ادمین دستی انجام نشود؛ همه چیز از این اسکریپت‌ها بیاید.

۱) infra/keycloak/config/: تعریف اعلانی (YAML یا JSON) به‌عنوان منبع حقیقت:
- ریلم crisis-ops:
  نقش‌های ریلم: national_admin، national_manager، provincial_manager، local_manager،
  crisis_operator، data_officer، system_admin، و نقش فنی audit-writer.
  Group های سلسله‌مراتبی با ویژگی geo_scope: /national، /province/tehran،
  /province/markazi، /province/hormozgan و زیرگروه‌های city هر استان
  (فهرست شهرها از فایل config/geo.yaml تا کارفرما بتواند تکمیل کند).
  کلاینت‌ها: crisis-web (public، PKCE S256، redirect URI و web origin دقیق، بدون wildcard)،
  crisis-api (confidential، فقط برای اعتبارسنجی و audience)، crisis-worker
  (service account برای ارتباط سرویس به سرویس، نقش audit-writer)،
  crisis-audit-reader (service account فقط با view-events).
  Protocol Mapper: نقش‌های ریلم، geo_scope از Group، audience=crisis-api.
- ریلم crisis-citizen: ثبت‌نام آزاد، تأیید ایمیل یا تلفن، کلاینت citizen-pwa
  (public، PKCE)؛ هیچ نقش مدیریتی؛ توکن بدون داده شخصی اضافه.
- سیاست‌ها در هر دو ریلم: password policy (طول ≥ ۱۲، پیچیدگی، عدم تکرار ۵ رمز قبلی،
  عدم استفاده از نام کاربری)، Brute Force Detection فعال، access token ≤ ۱۵ دقیقه،
  SSO idle و max مصوب، refresh token rotation، SSL required=all در production.
- OTP اجباری: Required Action برای همه اعضای نقش‌های manager، admin و data_officer،
  و یک Authentication Flow شرطی که برای این نقش‌ها OTP را الزامی کند.
- رویدادها در هر سه ریلم (master، crisis-ops، crisis-citizen): eventsEnabled،
  adminEventsEnabled، adminEventsDetailsEnabled، فهرست کامل نوع رویدادهای کاربر
  (LOGIN، LOGIN_ERROR، LOGOUT، CODE_TO_TOKEN، REFRESH_TOKEN، UPDATE_PASSWORD،
  UPDATE_TOTP، REMOVE_TOTP و ...)، eventsExpiration طبق سیاست نگهداری از متغیر محیطی،
  Event Listener ها: jboss-logging و crisis-audit (از P1.1).
- تم crisis-fa برای صفحه ورود: فارسی، RTL، فونت Vazirmatn محلی، بدون منبع خارجی؛
  پیام‌های فارسی در messages_fa.properties.

۲) infra/keycloak/bootstrap.sh (bash با set -euo pipefail):
- با kcadm.sh داخل کانتینر Keycloak وارد ریلم master شود؛ نام کاربری و رمز ادمین فقط
  از متغیر محیطی یا فایل secret (هیچ مقدار پیش‌فرض در کد؛ AFTA-11).
- منتظر سالم شدن Keycloak بماند (health/ready با timeout).
- idempotent باشد: هر شیء را اگر وجود دارد به‌روزرسانی و اگر ندارد ایجاد کند؛
  اجرای دوم هیچ تغییری ندهد و خطا ندهد.
- ورودی: ENV=dev|test|prod. در dev و test کاربران آزمایشی برای هر نقش و هر استان
  بسازد (رمز تصادفی، چاپ در فایل .secrets محلی که در .gitignore است)؛ در prod هرگز
  کاربر آزمایشی نسازد.
- client secret های confidential را بسازد یا بچرخاند و در Vault یا Kubernetes Secret
  (بر اساس ENV) بنویسد؛ هرگز در خروجی استاندارد چاپ نکند.
- ریلم master را سخت کند: غیرفعال کردن ثبت‌نام، OTP برای ادمین، رویدادها فعال.
- در پایان خروجی partial export هر ریلم (بدون secret و کاربر) را در
  infra/keycloak/exports/ بنویسد تا تفاوت‌ها در بازبینی PR دیده شود.
- اسکریپت infra/keycloak/verify.sh: با API ادمین بررسی کند که پیکربندی فعلی با
  config یکی است (drift detection) و در صورت تفاوت با کد خروج غیرصفر تمام شود.

۳) یکپارچگی:
- سرویس keycloak-bootstrap در docker-compose و Job همین اسکریپت در Helm (برای P6.3).
- مرحله CI: Keycloak در testcontainer بالا بیاید، bootstrap.sh دو بار اجرا شود و
  verify.sh پاس شود.
- docs/keycloak.md: راهنمای اجرا، افزودن استان یا شهر جدید (فقط با ویرایش geo.yaml و
  اجرای مجدد اسکریپت)، چرخش secret.

معیار پذیرش:
- `make keycloak-bootstrap` روی محیط خالی هر دو ریلم را کامل می‌سازد.
- اجرای دوم بدون تغییر و بدون خطا است؛ verify.sh پاس می‌شود.
- توکن کاربر آزمایشی provincial_manager تهران شامل نقش و geo_scope=province:tehran است.
- کاربر manager بدون OTP نمی‌تواند وارد شود.
- ورود ناموفق در رویدادهای کاربر و ایجاد کاربر در رویدادهای ادمین ریلم دیده می‌شود.
- در ENV=prod هیچ کاربر آزمایشی ساخته نمی‌شود.
```

### فاز ۱

#### P1.1 — مجوزدهی و Audit مبتنی بر Keycloak (T11)

```text
SRS: NFR-09، NFR-11، NFR-12. پلن: بخش ۱-۳، AFTA-04، AFTA-05، AFTA-07.
پیش‌نیاز: P0.5. ریلم‌ها و کلاینت‌ها را تغییر نده مگر از طریق config و bootstrap.sh.

۱) افزونه infra/keycloak/crisis-audit-spi (Java 17، Maven، وابستگی‌های Keycloak با
   scope=provided و نسخه برابر Keycloak مستقر):
- RealmResourceProvider با مسیر /realms/{realm}/crisis-audit/events:
  POST فقط با bearer token کلاینتی که نقش audit-writer دارد؛ بدنه شامل
  event_id (برای idempotency)، occurred_at، actor_sub، actor_session_id، action،
  resource_type، resource_id، geo_scope، outcome، details (محدود به ۸KB).
  ورودی سخت‌گیرانه اعتبارسنجی شود؛ event_id تکراری نادیده گرفته شود.
- EventListenerProvider به نام crisis-audit که رویدادهای کاربر، ادمین و رویدادهای
  کسب‌وکاری را در جدول append-only crisis_audit_log (در DB کیکلاک، با Liquibase یا
  JPA entity provider) با زنجیره هش ذخیره کند: hash = SHA-256(prev_hash + رکورد کانونیک).
- ارسال همزمان هر رکورد به SIEM کارفرما با syslog (RFC 5424) یا CEF روی TLS؛
  مقصد از تنظیمات SPI.
- GET /realms/{realm}/crisis-audit/events با فیلتر (کاربر، اقدام، منبع، بازه زمانی،
  geo) و صفحه‌بندی، فقط برای نقش view-events؛ و endpoint verify برای بررسی زنجیره هش.
- تست واحد جاوا و تست یکپارچگی با Keycloak در testcontainer.
- build در CI و کپی jar در image سفارشی Keycloak (infra/keycloak/Dockerfile)؛
  bootstrap.sh listener را در ریلم‌ها فعال کند.

۲) ماژول iam در بک‌اند:
- اعتبارسنجی JWT ریلم‌های crisis-ops و crisis-citizen با JWKS کش‌شده و چرخش کلید؛
  بررسی iss، aud=crisis-api، exp، و azp مجاز.
- مدل Pydantic به نام Principal: sub، session_id (sid)، realm، roles، geo_scope.
- وابستگی require_permission(permission, geo) که نقش و محدوده جغرافیایی را سمت سرور
  بررسی کند؛ نگاشت نقش به مجوز در یک جای واحد (permissions.py).
- قاعده تفکیک وظایف (AFTA-05) به‌صورت تابع مشترک برای ماژول‌های workflow و rules.

۳) ماژول audit در بک‌اند (بدون جدول Audit در DB برنامه):
- AuditClient.record(action, resource, outcome, details) که با توکن service account
  crisis-worker به endpoint افزونه ارسال کند؛ کاربر و session از Principal درخواست.
- Outbox محلی: اگر Keycloak در دسترس نبود، رکورد در جدول audit_outbox بماند و
  worker با retry ارسال کند؛ پس از تأیید، رکورد Outbox حذف شود. هیچ اقدامی بدون
  رکورد Audit یا Outbox commit نشود (در همان تراکنش).
- API: GET /api/v1/audit که با کلاینت crisis-audit-reader از Keycloak بخواند و
  رویدادهای کاربر، ادمین و کسب‌وکاری را در یک نمای واحد برگرداند
  (فقط system_admin و national_admin).
- فرانت: صفحه Audit با فیلتر و نمایش وضعیت زنجیره هش.

معیار پذیرش:
- کاربر استان مرکزی به منبع تهران 403 می‌گیرد و این رد دسترسی در Audit کیکلاک ثبت می‌شود.
- ورود ناموفق، ایجاد کاربر و یک اقدام کسب‌وکاری هر سه در صفحه Audit دیده می‌شوند.
- دستکاری مستقیم یک ردیف در crisis_audit_log با endpoint verify کشف می‌شود.
- با Keycloak خاموش، اقدام کسب‌وکاری انجام و در Outbox می‌ماند؛ پس از روشن شدن ارسال می‌شود.
- event_id تکراری دو رکورد نمی‌سازد.
```

#### P1.2 — پرونده بحران و تریگر دستی (T1، T10)

```text
SRS: FR-01.1، FR-02.1، FR-17.1 و بخش ۶-۱ (موجودیت Crisis Case). SAS: Crisis Core و Lifecycle.
- ماژول crisis: مدل CrisisCase (نوع، شدت، geometry PostGIS، زمان وقوع، گستره،
  سطح ملی/استانی/محلی، وضعیت، منشأ تریگر). State Machine با وضعیت‌های
  draft، active، under_assessment، monitoring، closed، archived و انتقال‌های مجاز
  صریح؛ هر انتقال غیرمجاز خطا بدهد.
- API: ایجاد بحران دستی، ثبت/ویرایش تحلیل اولیه، فهرست با فیلتر و صفحه‌بندی
  (فقط در محدوده جغرافیایی کاربر)، تغییر وضعیت توسط مدیر مجاز.
- پس از هر تغییر، رویداد crisis.triggered یا crisis.updated با الگوی Outbox
  (جدول outbox در همان تراکنش + انتشاردهنده به Kafka).
- همه عملیات با AuditClient در Audit کیکلاک ثبت شود (P1.1).
- فرانت: صفحه فهرست بحران‌ها، فرم ثبت بحران با انتخاب موقعیت روی نقشه
  (MapLibre با کاشی محلی)، فرم خلاصه اولیه.
معیار پذیرش: تست State Machine برای همه انتقال‌ها؛ تست Outbox (رویداد فقط پس از commit)؛
سناریوی Playwright ثبت بحران زلزله تهران.
```

#### P1.3 — Source Registry و کانکتورهای پایه (T2)

```text
SRS: FR-05.1، TR-T2.1، IF-01 تا IF-03. SAS: Connector Service.
- ماژول sources: مدل Source با همه فیلدهای TR-T2.1 (مالک، اتصال، سطح دسترسی،
  تناوب، قالب، سطح اعتماد، مسئول تأیید، سیاست نگهداری، وضعیت سلامت).
  منبع ثبت‌نشده یا تأییدنشده نباید داده‌اش به پردازش برسد.
- چارچوب کانکتور: کلاس پایه انتزاعی BaseConnector با متدهای fetch و health؛
  پیاده‌سازی‌ها: FileUploadConnector (CSV و Excel؛ بررسی نوع و اندازه فایل)،
  RestPollingConnector (httpx، timeout، retry با backoff)، DatabaseConnector
  (فقط خواندنی)، RssConnector، ManualEntry.
- هر رکورد دریافتی با source_id، received_at و ingest_id به لایه Raw در MinIO
  و رویداد data.ingested به Kafka برود.
- اعتبارنامه‌های منابع رمزنگاری‌شده ذخیره شوند (AFTA-10، AFTA-11).
- فرانت: صفحه ثبت و مدیریت منبع، آپلود فایل، وضعیت سلامت.
معیار پذیرش: تست هر کانکتور با سرور mock؛ تست رد داده منبع تأییدنشده؛ آپلود فایل غیرمجاز رد شود.
```

#### P1.4 — ستون رویداد Kafka (T3 پایه)

```text
SAS: بخش ۴ (جدول Topic ها) و ADR-02.
- core/kafka: تولیدکننده و مصرف‌کننده async (aiokafka) با سریال‌سازی Pydantic.
- events/: مدل Pydantic نسخه‌دار برای همه Topic های SAS (crisis.triggered،
  data.ingested، data.curated، data.quarantined، crisis.assessed،
  message.drafted، message.approved، message.delivered، crisis.updated،
  crisis.closed، source.health) با فیلدهای مشترک event_id، schema_version،
  occurred_at، trace_id.
- مصرف‌کننده idempotent (جدول processed_events)، retry محدود و ارسال به
  Topic جداگانه DLQ با علت خطا.
- اسکریپت ایجاد Topic ها با تعداد پارتیشن و replication قابل‌تنظیم.
- workers/: نقطه ورود عمومی اجرای مصرف‌کننده.
معیار پذیرش: تست یکپارچگی با testcontainers Kafka: پیام تکراری دوبار پردازش نشود؛
پیام خراب به DLQ برود.
```

#### P1.5 — فرانت پایه عملیاتی

```text
- صفحه داشبورد اولیه کارشناس: فهرست بحران‌های فعال در محدوده کاربر، آخرین رویدادها.
- کامپوننت مشترک جدول (صفحه‌بندی سمت سرور، مرتب‌سازی، فیلتر) و فرم.
- مدیریت خطای سراسری که trace_id را به کاربر نشان دهد.
- دسترس‌پذیری: ناوبری با صفحه‌کلید، برچسب‌های aria (jsx-a11y بدون خطا).
معیار پذیرش: تست Vitest کامپوننت‌ها؛ Lighthouse accessibility ≥ ۹۰ روی صفحات اصلی.
```

### فاز ۲

#### P2.1 — ETL و کیفیت پایه (T4)

```text
SRS: FR-06.1، TR-T4.1، TR-T4.2، DR-01. SAS: بخش ۵.
- ماژول quality: مصرف data.ingested، اعتبارسنجی Schema هر منبع (مدل Pydantic
  پویا یا JSON Schema ثبت‌شده در Source Registry)، فیلدهای اجباری، تاریخ معتبر،
  مختصات در محدوده ایران.
- لایه Curated در PostgreSQL؛ داده ناقص به Quarantine با علت (حذف نشود).
- Lineage: هر رکورد Curated شناسه ingest_id و فهرست تبدیل‌ها را نگه دارد.
- قواعد کیفیت قطعی قابل تعریف در جدول quality_rules.
- API و صفحه بازبینی قرنطینه برای data_officer: مشاهده، اصلاح، آزادسازی یا رد (ثبت در Audit کیکلاک).
معیار پذیرش: تست برای هر قاعده؛ رکورد با مختصات خارج از ایران به قرنطینه برود؛
مسیر Lineage از رکورد Curated تا فایل Raw قابل پیمایش باشد.
```

#### P2.2 — Rule Engine نسخه ۱ (T5)

```text
SRS: FR-04.1، FR-07.1، FR-08.1، TR-T5.1. SAS: Rule Engine و ADR-03.
- ماژول rules: قاعده به‌صورت داده (JSON اعتبارسنجی‌شده با Pydantic)، نه کد اجرایی؛
  هیچ eval یا exec. عملگرها: مقایسه، بازه، فاصله مکانی، پنجره زمانی، and/or.
- نسخه‌بندی قواعد؛ هر نسخه وضعیت draft، tested، active دارد. فعال‌سازی نیاز به
  تأیید نفر دوم دارد (AFTA-05، تابع تفکیک وظایف P1.1)؛ ایجاد، تست و فعال‌سازی هر نسخه
  در Audit کیکلاک ثبت شود.
- ارزیاب: ورودی رخداد و داده Curated، خروجی شدت، محدوده، نوع اثر و وضعیت
  significant/not_significant به‌همراه فهرست قواعد فعال‌شده (دلایل).
- قواعد نمونه سناریوی زلزله: بزرگی، عمق، فاصله از مراکز جمعیتی (آستانه‌ها قابل تنظیم).
- رویداد crisis.assessed.
- فرانت: صفحه نمایش ارزیابی با دلایل.
معیار پذیرش: تست جدول‌محور برای قواعد زلزله؛ تست اینکه قاعده بدون تأیید دوم فعال نشود.
```

#### P2.3 — گردش کار پایه (T7)

```text
SRS: FR-09.1، FR-10.1، FR-12.1، FR-13.1، TR-T7.1. SAS: Workflow Engine.
- ماژول workflow: صف کار (WorkItem) با اولویت و مالک؛ پرونده بحران‌های
  significant خودکار به صف استان مربوط برود.
- پیش‌نویس پیام (Message) با منشأ model/operator/template، سطح ۱/۲/۳، مخاطب،
  منطقه، کانال، فوریت.
- مسیر تأیید پیکربندی‌پذیر بر اساس سطح: سطح ۱ بدون تأیید، سطح ۲ یک تأیید، سطح ۳
  زنجیره کارشناس → مدیر استانی → مسئول کشوری. نویسنده نمی‌تواند تأییدکننده باشد.
- تأییدکننده هر مرحله با نقش ریلم و Group جغرافیایی Keycloak تعیین شود (مثلاً
  provincial_manager در /province/tehran)؛ فهرست تأییدکنندگان واجد شرایط از
  API ادمین Keycloak با کلاینت crisis-worker خوانده و کش شود. هیچ فهرست کاربر
  موازی در DB برنامه نگه داشته نشود.
- عملیات تأیید، رد، بازگشت برای اصلاح، با ثبت زمان پاسخ و توضیح؛ هر عمل با
  session_id کیکلاک در Audit ثبت شود.
- در این فاز انتشار واقعی نداریم؛ فقط رویداد message.approved.
- فرانت: صندوق کار کارشناس، صفحه بررسی پیش‌نویس، صفحه تأیید.
معیار پذیرش: تست همه مسیرها؛ تست منفی: پیام سطح ۳ با یک تأیید به approved نرسد.
```

#### P2.4 — قالب‌ها و دستورالعمل کلی

```text
SRS: FR-03.1، FR-20.
- کتابخانه قالب پیام و دستورالعمل با متغیرهای مجاز (مثلاً {region}، {time})،
  نسخه‌بندی و وضعیت تأیید. فقط قالب تأییدشده قابل استفاده است.
- انتخاب دستورالعمل کلی برای بحران و نمایش در پنل کارشناس.
- رندر امن قالب (بدون اجرای کد؛ escape کامل).
معیار پذیرش: تست رندر و تست تزریق در متغیرها.
```

### فاز ۳ (MVP)

#### P3.1 — NLP و RAG پایه (T6)

```text
SRS: FR-09.2 (بخش پیش‌نویس)، TR-T6.2، NFR-13. SAS: بخش ۶. پلن: AFTA-01، AFTA-16.
- ماژول nlp با رابط LlmClient برای سرور محلی سازگار با OpenAI (vLLM)؛
  آدرس از config؛ هیچ endpoint خارجی.
- پیش‌پردازش فارسی با Hazm؛ embedding با ParsBERT (مدل از مخزن داخلی بارگذاری شود)؛
  ذخیره در pgvector با metadata (نوع سند، استان، تاریخ اعتبار، وضعیت تأیید).
- بارگذاری فقط اسناد تأییدشده (دستورالعمل‌ها، قالب‌ها)؛ قطعه‌بندی و ایندکس.
- تولید پیش‌نویس پیام: بازیابی با فیلتر metadata، prompt با دستور صریح
  «فقط از منابع داده‌شده»، خروجی ساخت‌یافته Pydantic شامل متن، منابع و confidence.
- Guardrail: رد پاسخ بدون منبع، confidence زیر آستانه، یا تشخیص prompt injection
  در ورودی؛ وضعیت needs_review.
- ثبت کامل ورودی، منابع بازیابی‌شده، نسخه مدل و خروجی (NFR-14).
معیار پذیرش: تست با LLM جعلی (fake) برای Guardrail؛ پیش‌نویس همیشه منبع دارد یا needs_review است.
```

#### P3.2 — Dispatcher نسخه ۱ (T8)

```text
SRS: FR-11.1، FR-14.1، IF-07، IF-08.
- ماژول dispatch: مصرف message.approved؛ رابط ChannelAdapter با send و status.
- آداپتورها: WebFeedAdapter (انتشار در API عمومی و پورتال)، SmsTestAdapter
  (اتصال آزمایشی به درگاه کارفرما با پیکربندی).
- جدول Delivery با تلاش‌ها، وضعیت و رسید؛ retry با backoff؛ رویداد message.delivered.
- Rate limit به‌ازای کانال (Redis).
- انتشار سطح ۱ فقط از قالب‌های کم‌ریسک مصوب و فقط در محیط آزمایشی.
- endpoint عمومی خواندنی هشدارهای فعال برای پورتال (کش‌شده، با rate limit؛ AFTA-15).
معیار پذیرش: تست retry و ثبت وضعیت؛ تست اینکه پیام تأییدنشده هرگز ارسال نشود.
```

#### P3.3 — داشبورد، نقشه و چرخه عمر پایه (T9، T10)

```text
SRS: FR-15.1، FR-16.1، FR-17.1، FR-18.1، FR-19.1.
- داشبورد فنی: وضعیت منابع، رخدادها، خطاها و صف DLQ؛ نقشه MapLibre با موقعیت بحران‌ها
  (لایه GeoJSON از API).
- ثبت دستی گزارش جدید روی پرونده و تاریخچه کامل تغییرات (جدول نسخه).
- بستن دستی توسط مدیر مجاز؛ بایگانی پرونده، پیام‌ها و تصمیم‌ها؛ جستجوی ساده.
- پایش زمان‌بندی‌شده منابع (health) و رویداد source.health.
- اتصال به GIS کارفرما فقط ارسال و دریافت داده (طبق SRS)؛ رابط GisClient با پیاده‌سازی mock.
معیار پذیرش: تست API ها؛ Playwright: مشاهده بحران روی نقشه، ثبت گزارش، بستن، جستجو در بایگانی.
```

#### P3.4 — آماده‌سازی و پذیرش MVP

```text
- داده و سناریوی دمو زلزله (seed) برای سه استان با Mock API منابع.
- تست Playwright سناریوی کامل: تریگر → جمع‌آوری → کیفیت → ارزیابی → ارجاع →
  پیش‌نویس RAG → تأیید → انتشار وب/SMS آزمایشی → داشبورد → بستن → بایگانی.
- چک‌لیست پذیرش M4 در docs/acceptance/M4.md با ستون نیازمندی، روش آزمون، نتیجه
  برای همه نیازمندی‌های «.1».
- رفع همه هشدارهای ruff، mypy، eslint و یافته‌های امنیتی باز.
معیار پذیرش: سناریوی کامل سبز؛ چک‌لیست M4 کامل.
```

### فاز ۴

#### P4.1 — Stream worker و تشخیص خودکار (T3)

```text
SRS: FR-01.2، NFR-01. SAS: Stream Processor.
- استقرار جداگانه workers/stream (image مستقل).
- پنجره زمانی (tumbling و sliding)، حذف تکرار با کلید و Redis، checkpoint
  (commit آفست پس از پردازش موفق).
- هم‌بستگی چند منبع: اگر دو منبع مستقل در پنجره و فاصله مکانی تعریف‌شده
  نشانه زلزله دهند، بحران خودکار با منشأ auto ایجاد شود و رویداد تکراری حذف شود.
- معیار تأخیر: متریک Prometheus برای فاصله occurred_at تا هشدار.
معیار پذیرش: تست بار با تولیدکننده مصنوعی: p95 کمتر از ۵۰۰ms؛ تست حذف تکرار.
```

#### P4.2 — کیفیت پیشرفته و NER (T4، T6)

```text
SRS: FR-06.2، TR-T4.1، TR-T4.2.
- پاک‌سازی متن فارسی (نرمال‌سازی، حذف تکرار تقریبی با MinHash یا embedding).
- NER برای سازمان، تاریخ، نوع بحران و منطقه (spaCy pipeline با مدل fine-tuned؛
  مسیر آموزش و ارزیابی در nlp/training).
- امتیاز کیفیت چندشاخصه (کامل‌بودن، تازگی، سازگاری بین منابع، اعتبار مکان و زمان،
  اعتماد منبع) و آستانه قرنطینه قابل تنظیم.
- قواعد آماری ناهنجاری در Batch (نرخ خطا، تغییر الگو).
- داشبورد کیفیت: وضعیت منابع، نرخ خطا، تعداد قرنطینه، تاریخچه اصلاحات.
معیار پذیرش: گزارش ارزیابی NER روی مجموعه آزمون؛ تست امتیاز کیفیت.
```

#### P4.3 — کانکتورهای جریانی و DLQ (T2)

```text
SRS: FR-05.2، TR-T2.2، IF-04.
- کانکتور جریانی برای منابع بلادرنگ و فید شبکه‌های اجتماعی تأمین‌شده توسط کارفرما.
- fallback ingestion با Mock API و CSV Proxy.
- رابط بازبینی DLQ: مشاهده، بازپخش، رد؛ بازبینی خودکار پس از دوره بحران.
- هشدار قطع منبع و بازیابی خودکار جریان.
معیار پذیرش: تست قطع منبع: پس از بازگشت، هیچ رکوردی از دست نرفته است.
```

#### P4.4 — هم‌بستگی، امتیاز ریسک و Nowcasting (T5)

```text
SRS: FR-07.2، FR-08.2. SAS: ML / Nowcasting.
- هم‌بستگی مکانی-زمانی منابع (PostGIS) و تشخیص اثرات ثانویه با Rule + مدل اولیه.
- Feature Store ساده (جدول ویژگی‌ها با نسخه).
- مدل پایه امتیاز ریسک (مثلاً gradient boosting) با رجیستری نسخه مدل؛
  خروجی امتیاز، شواهد، confidence.
- مسیریابی خودکار: confidence بالا و significant به صف کارشناس، پایین به
  needs_review، not_significant به بایگانی.
- امکان غیرفعال‌سازی مدل و بازگشت به نسخه قبلی (NFR-14).
معیار پذیرش: تست مسیریابی؛ تست rollback مدل.
```

### فاز ۵

#### P5.1 — مدیریت قواعد و پیشنهاد مسیر (T5، T7)

```text
SRS: FR-04.2، FR-10.2.
- پنل مدیریت Rule: ایجاد، نسخه‌بندی، اجرای آزمایشی روی داده تاریخی،
  مقایسه نسخه‌ها، فرمول‌های تعیین ریسک برای چند نوع بحران؛ بدون نیاز به کدنویسی (NFR-15).
- پیشنهاد خودکار سطح اهمیت و مسیر تأیید بر اساس قواعد و نوع بحران.
معیار پذیرش: تست اجرای آزمایشی؛ Playwright ایجاد و فعال‌سازی قاعده با تأیید دوم.
```

#### P5.2 — چت‌بات شهروندی (T6)

```text
SRS: FR-09.2، TR-T6.1، TR-T6.2، NFR-02، NFR-13. پلن: AFTA-16.
- Intent detection، slot filling و تشخیص زبان (فارسی اولویت ۱؛ ترکی و عربی اولویت ۲).
- پاسخ RAG فقط از اسناد تأییدشده با نمایش منبع؛ امتناع در موارد بخش ۶-۲ SAS.
- تشخیص درخواست اضطراری و ارجاع به اپراتور (یا اورژانس طبق دستورالعمل کارفرما).
- رفت‌وبرگشت کارشناس و چت‌بات.
- ابزار ارزیابی evals/chatbot: مجموعه آزمون مرجع با دسته‌های بخش ۷-۱ SRS؛
  محاسبه شاخص‌های جدول ۱۰ (صحت، Groundedness، پوشش استناد، نرخ توهم، صحت ارجاع،
  صحت امتناع، زمان پاسخ) و اجرا در CI روی هر تغییر prompt یا مدل.
- محافظت در برابر prompt injection و نشت دستور سیستم؛ rate limit به‌ازای کاربر.
معیار پذیرش: Intent ≥ ۹۰٪، F1 جستجو ≥ ۰٫۸۵، زمان پاسخ p95 کمتر از ۳ ثانیه.
```

#### P5.3 — سه سطح تأیید عملیاتی (T7)

```text
SRS: FR-11.2، FR-12.2، FR-13.2، FR-20، FR-21.
- انتشار خودکار سطح ۱ فقط از قالب‌های مصوب کم‌ریسک.
- سطح ۲: تأیید مدیر محلی یا استانی با SLA؛ Escalation خودکار در صورت عبور از SLA
  به Group والد در ساختار جغرافیایی Keycloak (شهر → استان → ملی).
- تأیید پیام سطح ۲ و ۳ نیاز به step-up authentication دارد: توکن باید با OTP در همان
  session صادر شده باشد (بررسی acr یا amr در توکن)؛ در غیر این صورت کاربر به
  احراز مجدد با OTP هدایت شود.
- سطح ۳: زنجیره کامل؛ انتشار خودکار پیام حساس ممنوع؛ همه شرایط FR-21 بررسی شود.
- دکمه توقف اضطراری انتشار (سراسری و به‌ازای پیام)، فقط برای نقش‌های مجاز ریلم crisis-ops و ثبت در Audit کیکلاک.
- نمایش منشأ هر پیام (مدل، اپراتور، قالب).
معیار پذیرش: تست‌های منفی گسترده برای سطح ۳؛ تست Escalation با زمان مصنوعی.
```

#### P5.4 — کانال‌های انتشار (T8)

```text
SRS: FR-03.2، FR-14.2، TR-T8.1، IF-08، IF-09.
- آداپتورهای SMS، Push (Web Push برای PWA)، Email، IVR و پیام‌رسان‌های مورد تأیید
  (بله، ایتا، و سایر موارد در چارچوب مجاز) پشت رابط ChannelAdapter.
- انتخاب کانال جایگزین در صورت قطع کانال.
- رسید تحویل و گزارش نرخ تحویل به‌ازای کانال.
معیار پذیرش: تحویل ≥ ۹۵٪ در تست با شبیه‌ساز درگاه؛ حداقل ۴ کانال فعال.
```

#### P5.5 — خبر مشکوک و پورتال شهروندی (T9)

```text
SRS: FR-22، FR-23.
- گردش کار ارزیابی خبر: ثبت منبع و مسیر، امتیاز اعتبار، مقایسه با منابع رسمی،
  طبقه‌بندی شش‌گانه FR-22؛ برچسب «خبر جعلی» فقط با تأیید مرجع مجاز.
- PWA شهروندی با ریلم crisis-citizen و کلاینت citizen-pwa: مشاهده هشدارها بدون ورود؛
  ورود فقط برای ارسال گزارش و پیگیری. تأیید شماره تلفن با OTP پیامکی نیاز به
  Authenticator سفارشی Keycloak متصل به درگاه SMS کارفرما دارد؛ آن را در همان
  پروژه SPI (P1.1) پیاده کن و در bootstrap.sh فعال کن.
- PWA شهروندی: وضعیت هشدارهای منطقه، گزارش مردمی با عکس و موقعیت
  (حذف EXIF، محدودیت اندازه، اسکن فایل)، پیگیری تیکت، نوتیفیکیشن.
- داده شخصی شهروند رمزنگاری‌شده و کمینه (AFTA-10، NFR-11).
معیار پذیرش: تست طبقه‌بندی؛ Playwright ارسال گزارش از موبایل (viewport موبایل).
```

### فاز ۶

#### P6.1 — داشبورد عملیاتی (T9)

```text
SRS: FR-15.2، TR-T9.1.
- داشبورد سلسله‌مراتبی شهری، استانی، ملی با Drill-down و KPI پایه
  (زمان اطلاع‌رسانی، نرخ تحویل، زمان تأیید، وضعیت منابع).
- لایه رویداد بلادرنگ روی نقشه (WebSocket یا SSE).
- استخراج Polygon از متن (موقعیت‌های NER) و نمایش روی نقشه.
- خروجی PDF، Excel و تصویر.
معیار پذیرش: Playwright Drill-down از ملی تا شهر؛ تست خروجی‌ها.
```

#### P6.2 — چرخه عمر کامل (T10)

```text
SRS: FR-16.2، FR-17.2، FR-18.2، DR-02.
- به‌روزرسانی خودکار پرونده با داده جدید و اجرای مجدد ارزیابی (نودهای ۴ و ۵).
- پیشنهاد ادامه یا خاتمه بر اساس آخرین داده؛ نیاز به تأیید مدیر.
- تفکیک بایگانی نهایی و پایش نهفته؛ سیاست نگهداری؛ کنترل دسترسی به بایگانی.
- ثبت درس‌آموخته (Lesson Learned) و جستجو در آن.
معیار پذیرش: تست اجرای مجدد ارزیابی؛ تست سیاست نگهداری.
```

#### P6.3 — پایش ۷×۲۴ و تاب‌آوری (T12)

```text
SRS: FR-19.2، NFR-06، NFR-07.
- Prometheus، Grafana، Loki، OpenTelemetry (self-hosted)؛ داشبورد و هشدار برای
  تأخیر، خطا، صف‌ها، سلامت منابع.
- Helm chart با HPA، PodDisruptionBudget، Kafka با replication، PostgreSQL با
  replica و پشتیبان رمزنگاری‌شده؛ مستند DR با RPO و RTO.
- Keycloak در حالت cluster (حداقل دو نمونه، کش توزیع‌شده)، پایگاه داده جدا با replica،
  Job اجرای bootstrap.sh و verify.sh در هر استقرار، پشتیبان روزانه DB کیکلاک
  (شامل crisis_audit_log) و تست بازیابی ریلم‌ها.
- هشدار برای: افزایش LOGIN_ERROR، پر شدن audit_outbox، شکست ارسال به SIEM،
  و drift پیکربندی ریلم.
- تمرین Chaos (قطع پاد، قطع منبع، قطع کانال) و ثبت نتایج در docs/ops.
معیار پذیرش: گزارش تمرین Fail/Recover؛ تست بازیابی پشتیبان.
```

#### P6.4 — تست بار و کارایی

```text
SRS: NFR-01 تا NFR-05.
- سناریوهای k6 یا Locust: ۱۰۰۰ کاربر همزمان، پیک چت‌بات، پیک انتشار.
- گزارش p50، p95، p99 و گلوگاه‌ها؛ اصلاح موارد.
معیار پذیرش: همه اهداف NFR-01 تا NFR-04 برآورده شود؛ گزارش در docs/perf.
```

#### P6.5 — بسته آمادگی افتا

```text
پلن: بخش ۳ (AFTA-01 تا AFTA-24).
- docs/security/: سند معماری امنیت، مدل تهدید STRIDE برای هر ماژول،
  چک‌لیست OWASP ASVS سطح ۲ با وضعیت هر کنترل، رویه پاسخ به حادثه،
  ماتریس کنترل‌های AFTA به کد و تست.
- سخت‌سازی: image های non-root و حداقلی، NetworkPolicy برای تفکیک ناحیه‌ها،
  mTLS داخلی، CIS Benchmark.
- سخت‌سازی Keycloak (AFTA-24): کنسول ادمین و ریلم master فقط از شبکه مدیریت،
  production mode با TLS، غیرفعال بودن ویژگی‌های بی‌استفاده، بررسی redirect URI ها،
  مرور دوره‌ای کاربران و نقش‌ها با گزارش از API ادمین.
- اجرای DAST (OWASP ZAP) روی محیط تست؛ رفع یافته‌ها.
- آماده‌سازی محیط و مستندات برای Pen-Test آزمایشگاه مورد تأیید افتا.
معیار پذیرش: هیچ یافته High/Critical باز؛ همه ردیف‌های ماتریس AFTA شواهد دارند.
```

### فاز ۷

#### P7.1 — استقرار پایلوت و پذیرش

```text
SRS: بخش ۷-۱ (معیار پایان ماه ۸).
- values جداگانه Helm برای سه استان (تهران، مرکزی، هرمزگان) و سطح ملی.
- اجرای bootstrap.sh با ENV=prod؛ تکمیل geo.yaml با شهرهای نهایی؛ ایجاد کاربران
  واقعی و عضویت در Group ها از فایل تحویلی کارفرما با اسکریپت
  infra/keycloak/import_users.sh (بدون رمز؛ کاربر در اولین ورود رمز و OTP تنظیم کند).
- اسکریپت‌های UAT در docs/acceptance/M8.md برای همه نیازمندی‌های «.2».
- Runbook عملیاتی، راهنمای کاربری (مدیر، کارشناس، تأییدکننده، شهروند)،
  راهنمای مدیریت مدل و محتوای چت‌بات.
- گزارش پذیرش هر استان و گزارش شاخص‌های عملکرد و کیفیت.
معیار پذیرش: UAT سه استان امضا شده؛ همه اسناد تحویل بخش ۲-۷-۶ پروپوزال موجود است.
```

---

## ۹. موارد باز برای تأیید کارفرما

| مورد | تصمیم پیشنهادی | بخش |
| --- | --- | --- |
| ماژولار مونولیت تا ماه ۴ به‌جای ۱۴ سرویس | تأیید و ثبت ADR-08 | ۱-۱ |
| PWA به‌جای اپ بومی برای پایلوت | تأیید | ۱-۲ |
| چک‌لیست رسمی و سطح ارزیابی افتا، الگوریتم‌های رمز مجاز | دریافت از کارفرما | ۳ |
| حدود عددی شاخص‌های NFR-13 و ظرفیت NFR-05 | تعیین در سند پذیرش | SRS ۷-۱ |
| فهرست منابع، کانال‌ها و پیام‌رسان‌های مجاز هر استان | مرحله اول پایلوت | SRS ۲-۴ |
| مخزن آینه بسته‌ها و رجیستری image داخلی | تأمین توسط کارفرما | AFTA-02 |
| Keycloak به‌عنوان مرجع Audit کسب‌وکاری با افزونه SPI (ADR-09) | تأیید؛ یا جایگزین: Keycloak فقط برای رویدادهای هویتی و Audit کسب‌وکاری در سرویس جدا | ۱-۳ |
| درگاه SMS برای OTP شهروندان در Keycloak | تأمین API درگاه توسط کارفرما | P5.5 |
| زمان نگهداری رویدادهای Audit و مقصد SIEM/SOC | تعیین توسط کارفرما | P0.5، P1.1 |
