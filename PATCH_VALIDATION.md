# اعتبارسنجی وصله‌های امنیتی cJSON در CI

این مخزن کتابخانهٔ واقعی [cJSON](https://github.com/DaveGamble/cJSON) است، در commit
`826cd6f`. این commit والد رفع آسیب‌پذیری **heap buffer over-read** در `parse_object()`
(issue شمارهٔ 800 در مخزن بالادست) است. شاخهٔ `main` همین نسخهٔ آسیب‌پذیر است.
هر توسعه‌دهنده وصلهٔ خود را در یک شاخهٔ جدا push می‌کند و پایپ‌لاین GitHub Actions با ابزار
[patch-validator](https://github.com/mohammaderfanali/patch-validator) بررسی می‌کند که
وصله آسیب‌پذیری را کامل و درست رفع کرده است یا نه.

**آسیب‌پذیری:** وقتی یک object درست بعد از کاما تمام می‌شود (مثلاً `{"a":1,`)،
`parse_object()` بدون بررسی طول، بایت بعدی را می‌خواند.

## تغییرات نسبت به cJSON بالادست

| مسیر | کاربرد |
|------|--------|
| `validation/harness/cjson_cli.c` | رابط خط فرمان: ورودی را در بافری با اندازهٔ دقیق و بدون NUL به `cJSON_ParseWithLength()` می‌دهد |
| `validation/inputs/` | ورودی‌های آزمون (valid / boundary / invalid / security) |
| `validation/make_inputs.py` | تولیدکنندهٔ قطعی ورودی‌ها |
| `validation/run_functional.py` | اجراکنندهٔ سریع آزمون‌های عملکردی |
| `validation/SPECIFICATION.md` | استخراج Oracleها از RFC 8259 و مستندات cJSON |
| `.patch-validator/cjson.yaml` | پیکربندی و Oracleهای patch-validator |
| `.github/workflows/security-patch-ci.yml` | پایپ‌لاین کامل |

workflowهای CI بالادست (`CI.yml`، `ci-fuzz.yml`، Travis و AppVeyor) حذف شده‌اند تا فقط
همین پایپ‌لاین اجرا شود. کد خود کتابخانه دست نخورده است.

## شاخه‌های وصله

| شاخه | تغییر | نتیجهٔ مورد انتظار |
|------|-------|---------------------|
| `fix/upstream-bounds-check` | همان commit رفع بالادست (`3ef4e4e`): بررسی دسترسی‌پذیری بایت بعد از کاما | ✅ `PASS` (exit 0) |
| `fix/top-level-only` | همان بررسی، ولی فقط برای object سطح بالا (`depth == 1`) | ❌ `VULN_REMAINS` (exit 10): ورودی تودرتوی `{"outer":{"inner":1,` همچنان over-read دارد |
| `fix/lower-nesting-limit` | رفع درست + کاهش `CJSON_NESTING_LIMIT` از 1000 به 64 | ❌ `FUNC_REGRESSION` (exit 20): سند معتبر ۱۰۰ سطحی رد می‌شود |

## پایپ‌لاین

```
prepare ─┬─ static-analysis ───────────────────────────────┐
         ├─ build (gcc|clang × release|sanitized) ─┬─ functional-tests ─┤
         │                                         └─ patch-validator ──┤
         └──────────────────────────────────────────────────────────────┴─ verdict
```

| Job | کار | مسدودکننده؟ |
|-----|-----|-------------|
| `prepare` | تعیین `base` (merge-base با main) و `patched` (commit پوش‌شده)، تولید `patch.diff`، هشدار در صورت دست‌کاری فایل‌های مورد اعتماد | بله |
| `static analysis` | `cppcheck` و `gcc -fanalyzer` روی `cJSON.c` | خیر (اطلاعاتی) |
| `build` | ساخت harness و کتابخانه با gcc و clang، در حالت release و sanitized (ASan+UBSan)، با `-Werror` | بله |
| `functional tests` | اجرای باینری sanitized روی ورودی‌های valid/boundary/invalid و مقایسه با Oracle | بله |
| `patch-validator` | `doctor` → `check-config` → `hardening` → `validate` → بازاجرای suite رگرسیون → گزارش و artifact | بله |
| `verdict` | جمع‌بندی همهٔ نتایج در یک status check | بله |

**مرز اعتماد:** پیکربندی (`.patch-validator/`)، harness و ورودی‌ها (`validation/`) همیشه از
نسخهٔ پایه خوانده می‌شوند، نه از وصله.

## اجرای محلی

```bash
gcc -std=c99 -O1 -g -fsanitize=address,undefined -I. -o cjson_cli validation/harness/cjson_cli.c cJSON.c -lm
python3 validation/run_functional.py --binary ./cjson_cli
```

```bash
pip install git+https://github.com/mohammaderfanali/patch-validator
patch-validator validate --config .patch-validator/cjson.yaml --base-revision main --patched-revision fix/top-level-only
```
