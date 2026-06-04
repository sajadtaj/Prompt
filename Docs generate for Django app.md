## ساخت مستند برای هر Django App مستقل

محتوای زیر درون docgen.sh و دورن همان app جنگو باید قرار بگیرد وصدا زده شو.د.



```
#!/usr/bin/env bash
set -euo pipefail

# ============================================================
# docgen.sh - مستندساز خودکار برای اپ‌های جنگو
#
# اجرا: در ریشه یک اپ جنگو (مثل monitoring_modul/)
# ./docgen.sh
#
# خروجی: [نام اپ]_doc.md در همان دایرکتوری
# ============================================================

# ============================================================
# 🔧 تنظیمات کاربر (اینجا را ویرایش کن)
# ============================================================

# ✅ پسوندهایی که باید مستند شوند (بدون نقطه، با کاما از هم جدا شده)
#    مثال: py,html,sql,yml,yaml,json,conf
INCLUDE_EXTS=(
    "py"
    "html"
    "sql"
    "yml"
    "yaml"
    "json"
    "conf"
)

# ✅ نام فایل‌های خاص که باید مستند شوند (حتی اگر پسوند استاندارد ندارند)
INCLUDE_NAMES=(
    "Dockerfile"
    "env.example"
    "enc.example"
)

# ❌ الگوهایی که نباید مستند شوند (پسوند یا نام فایل)
#    می‌توانی از * استفاده کنی برای الگوی عمومی
EXCLUDE_PATTERNS=(
    "*.pyc"
    "*.pyo"
    "*.tmp"
    "*.log"
    "*.swp"
    "*~"
    "*.md"          # فایل مارک‌دون مستند نشود
"*.sh"          # فایل شل اسکریپت (از جمله خود docgen.sh) مستند نشود
"*.env"         # فایل‌های محیطی واقعی مستند نشوند
)

# ❌ نام فایل‌های خاص که نباید مستند شوند (دقیقاً نام فایل)
EXCLUDE_NAMES=(
    "docgen.sh"     # خود اسکریپت مستند نشود
)

# ❌ پوشه‌هایی که نباید وارد شوند
EXCLUDE_DIRS=(
    "__pycache__"
    "migrations"
    ".git"
    ".venv"
    "venv"
    "env"
    ".idea"
    ".vscode"
    "__pypackages__"
    ".pytest_cache"
    ".mypy_cache"
    ".ruff_cache"
)

# ============================================================
# رنگ‌ها برای خروجی ترمینال
# ============================================================
RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
NC='\033[0m' # No Color

# ============================================================
# توابع کمکی
# ============================================================

error() {
    echo -e "${RED}[ERROR]${NC} $1" >&2
exit 1
}

warn() {
    echo -e "${YELLOW}[WARN]${NC} $1" >&2
}

info() {
    echo -e "${GREEN}[INFO]${NC} $1"
}

# بررسی وجود ابزارهای لازم
check_dependencies() {
    if ! command -v tree &> /dev/null; then
error "'tree' پیدا نشد. نصب کنید: sudo apt install tree (Debian/Ubuntu) یا sudo yum install tree (RHEL/CentOS)"
fi
}

# بررسی اینکه آیا فایل با الگوی exclude مطابقت دارد
match_exclude_pattern() {
    local file="$1"
local filename=$(basename "$file")

# بررسی نام فایل در EXCLUDE_NAMES
for exclude_name in "${EXCLUDE_NAMES[@]}"; do
        if [[ "$filename" == "$exclude_name" ]]; then
return 0
fi
    done

# بررسی الگوهای EXCLUDE_PATTERNS (مثل *.pyc, *.tmp, ...)
for pattern in "${EXCLUDE_PATTERNS[@]}"; do
        if [[ "$filename" == $pattern ]]; then
return 0
fi
    done

return 1
}

# بررسی اینکه آیا پسوند فایل در لیست Include است
is_ext_included() {
    local file="$1"
local filename=$(basename "$file")

# ابتدا بررسی کن نام فایل در INCLUDE_NAMES هست؟ (مثل Dockerfile)
for include_name in "${INCLUDE_NAMES[@]}"; do
        if [[ "$filename" == "$include_name" ]]; then
return 0
fi
    done

# سپس پسوند را بررسی کن
local ext="${filename##*.}"
# اگر فایل پسوند نداشت (مثل Dockerfile) و در INCLUDE_NAMES هم نبود، رد کن
if [[ "$filename" == "$ext" ]]; then
return 1
fi

    for include_ext in "${INCLUDE_EXTS[@]}"; do
        if [[ "$ext" == "$include_ext" ]]; then
return 0
fi
    done

return 1
}

# بررسی اینکه آیا فایل باید مستند شود یا خیر (ترکیب هر دو شرط)
should_document_file() {
    local file="$1"

# اول: اگر با الگوهای exclude مطابقت داشت، رد کن
if match_exclude_pattern "$file"; then
return 1
fi

# دوم: اگر پسوند یا نام در Include بود، قبول کن
if is_ext_included "$file"; then
return 0
fi

# در غیر این صورت رد کن
return 1
}

# بررسی پوشه‌های exclude
should_exclude_dir() {
    local dirname="$1"

for exclude_dir in "${EXCLUDE_DIRS[@]}"; do
        if [[ "$dirname" == "$exclude_dir" ]]; then
return 0
fi
    done

return 1
}

# استخراج نام اپ از مسیر فعلی یا apps.py
get_app_name() {
    local current_dir
    current_dir=$(basename "$(pwd)")

if [[ -f "apps.py" ]]; then
local app_name
        app_name=$(grep -E "^\s*name\s*=\s*['\"]" apps.py | head -1 | sed -E "s/.*['\"]([^'\"]*)['\"].*/\1/")
if [[ -n "$app_name" ]]; then
echo "$app_name"
return
fi
    fi

echo "$current_dir"
}

# استخراج توضیح اپ از apps.py
get_app_description() {
    if [[ -f "apps.py" ]]; then
local desc
        desc=$(grep -E "^\s*verbose_name\s*=\s*['\"]" apps.py | head -1 | sed -E "s/.*['\"]([^'\"]*)['\"].*/\1/")
if [[ -n "$desc" ]]; then
echo "$desc"
return
fi
    fi

echo "ماژول $(basename "$(pwd)")"
}

# تولید ساختار درختی
generate_tree() {
    local exclude_pattern_str
    local exclude_dirs_str

    # ساخت الگوی tree برای exclude
exclude_dirs_str=$(IFS='|'; echo "${EXCLUDE_DIRS[*]}")
exclude_pattern_str="__pycache__|migrations|${exclude_dirs_str}"

# اضافه کردن الگوهای فایل به tree
for pattern in "${EXCLUDE_PATTERNS[@]}"; do
# تبدیل *.pyc به *.pyc (همانطور هست)
exclude_pattern_str="${exclude_pattern_str}|${pattern}"
done

local tree_output
    tree_output=$(tree -I "$exclude_pattern_str" --prune -F 2>/dev/null)

if [[ -z "$tree_output" ]]; then
warn "tree خروجی کامل نداد، از find استفاده می‌شود..."
echo "."
# ساخت exclude شرطی برای find
local find_exclude=""
for dir in "${EXCLUDE_DIRS[@]}"; do
find_exclude="$find_exclude ! -path \"./$dir/*\""
done
        for pattern in "${EXCLUDE_PATTERNS[@]}"; do
# تبدیل *.pyc به -name "*.pyc"
find_exclude="$find_exclude ! -name \"$pattern\""
done
eval "find . -type f $find_exclude | sed 's|^\./||' | sort | awk -F/ '{for(i=1;i<NF;i++){if(!seen[\$0]++)printf \"    \";}; print \"    \" \$NF}'"
else
echo "$tree_output"
fi
}

# جمع‌آوری لیست فایل‌های مهم
collect_important_files() {
    local files=()

    # ساخت شرط find برای exclude پوشه‌ها
local find_exclude_dirs=""
for dir in "${EXCLUDE_DIRS[@]}"; do
find_exclude_dirs="$find_exclude_dirs ! -path \"./$dir/*\""
done

# حلقه برای پیدا کردن فایل‌ها
while IFS= read -r file; do
        if should_document_file "$file"; then
files+=("$file")
        fi
    done < <(eval "find . -type f $find_exclude_dirs | sed 's|^\./||' | sort")

    printf '%s\n' "${files[@]}"
}

# تعیین آیکون بر اساس نوع فایل
get_file_icon() {
    local filename=$(basename "$1")

case "$filename" in
Dockerfile)               echo "🐳" ;;
*.py)                     echo "🧩" ;;
*.html)                   echo "🌐" ;;
*.sh)                     echo "🔧" ;;
*.yml|*.yaml)             echo "⚙️" ;;
*.sql)                    echo "🗄️" ;;
*.json)                   echo "📦" ;;
*.conf)                   echo "⚙️" ;;
*.env.example|*.enc.example) echo "🔐" ;;
*)                        echo "📄" ;;
    esac
}

# تعیین زبان برای بلاک کد Markdown
get_code_lang() {
    local filename=$(basename "$1")

case "$filename" in
Dockerfile)               echo "dockerfile" ;;
*.py)                     echo "python" ;;
*.html)                   echo "html" ;;
*.sh)                     echo "bash" ;;
*.yml|*.yaml)             echo "yaml" ;;
*.sql)                    echo "sql" ;;
*.json)                   echo "json" ;;
*.conf)                   echo "ini" ;;
*.env.example|*.enc.example) echo "bash" ;;
*)                        echo "text" ;;
    esac
}

# محاسبه تعداد خطوط فایل
count_lines() {
    local file="$1"
if [[ -f "$file" ]]; then
wc -l < "$file" 2>/dev/null | tr -d ' '
else
echo "0"
fi
}

# محاسبه حجم فایل
get_file_size() {
    local file="$1"
if [[ -f "$file" ]]; then
stat -c %s "$file" 2>/dev/null || stat -f %z "$file" 2>/dev/null || echo "0"
else
echo "0"
fi
}

# نمایش محتوای کامل فایل
show_file_content() {
    local file="$1"
local lang="$2"

echo "\`\`\`$lang"
cat "$file" 2>/dev/null || echo "[خطا در خواندن فایل]"
echo "\`\`\`"
}

# تولید فایل Markdown
generate_markdown() {
    local app_name="$1"
local app_desc="$2"
local output_file="$3"
shift 3
local important_files=("$@")

    info "ایجاد فایل: $output_file"

{
        echo "# $app_name"
echo ""
echo "**توضیح:** $app_desc"
echo ""
echo "---"
echo ""
echo "## 🌳 ساختار درختی پروژه"
echo ""
echo "\`\`\`text"
generate_tree
        echo "\`\`\`"
echo ""
echo "---"
echo ""
echo "## 📋 فهرست فایل‌های مهم"
echo ""

if [[ ${#important_files[@]} -eq 0 ]]; then
echo "*هیچ فایل مهمی پیدا نشد.*"
else
            for file in "${important_files[@]}"; do
echo "- \`$file\`"
done
        fi

echo ""
echo "---"
echo ""
echo "## 📄 محتوای فایل‌ها"
echo ""

local current_dir=""
for file in "${important_files[@]}"; do
local dir_path=$(dirname "$file")

if [[ "$dir_path" != "$current_dir" ]]; then
current_dir="$dir_path"
if [[ "$current_dir" == "." ]]; then
echo "### 📁 ریشه (root)"
else
echo "### 📁 $current_dir/"
fi
echo ""
fi

local icon=$(get_file_icon "$file")
            local lang=$(get_code_lang "$file")
            local lines=$(count_lines "$file")
            local size=$(get_file_size "$file")

            echo "#### $icon \`$file\`"
echo ""
echo "- **خطوط:** $lines"
echo "- **حجم:** $size bytes"
echo ""

show_file_content "$file" "$lang"
echo ""
done
} > "$output_file"
}

# ============================================================
# اجرای اصلی
# ============================================================

main() {
    info "شروع مستندسازی اپ جنگو..."

check_dependencies

if [[ ! -f "admin.py" ]] && [[ ! -f "models.py" ]] && [[ ! -f "apps.py" ]]; then
warn "به نظر نمی‌رسد در ریشه یک اپ جنگو باشید"
read -p "آیا ادامه می‌دهید؟ (y/N): " -n 1 -r
        echo
if [[ ! $REPLY =~ ^[Yy]$ ]]; then
error "اجرا لغو شد."
fi
    fi

APP_NAME=$(get_app_name)
APP_DESC=$(get_app_description)
OUTPUT_FILE="${APP_NAME}_doc.md"

info "نام اپ: $APP_NAME"
info "توضیح: $APP_DESC"
info "خروجی: $OUTPUT_FILE"

mapfile -t IMPORTANT_FILES < <(collect_important_files)
    info "${#IMPORTANT_FILES[@]} فایل مهم پیدا شد."

if [[ ${#IMPORTANT_FILES[@]} -eq 0 ]]; then
warn "هیچ فایل مهمی پیدا نشد."
fi

generate_markdown "$APP_NAME" "$APP_DESC" "$OUTPUT_FILE" "${IMPORTANT_FILES[@]}"

info "✅ مستندسازی کامل شد!"
info "فایل خروجی: $(pwd)/$OUTPUT_FILE"
}

main "$@"
```

---

## پرامپت بررسی معماری و کد  مستند تولید شده بالا


#### نحوه استفاده (گام به گام)

##### مرحله ۱: تولید `monitoring_module_doc.md` با اسکریپت خودت
```bash
cd /path/to/monitoring_module
./docgen.sh
# خروجی: monitoring_module_doc.md
```

##### مرحله ۲: کپی محتوای پرامپت بالا
- جای `[NAME]` را با نام واقعی اپ عوض کن (مثل `monitoring_module`)

##### مرحله ۳: ارسال به همراه فایل
- پرامپت را به همراه فایل `monitoring_module_doc.md` برای من (یا هر هوش مصنوعی دیگری) ارسال کن

##### مرحله ۴: دریافت خروجی
- تو فایل‌های `ARCHITECTURE.md` و `CODE_REVIEW.md` را به صورت کامل دریافت می‌کنی

---


#### پرامپت نهایی و استاندارد (برای کپی و استفاده)

```markdown
# نقش: تو یک معمار نرم‌افزار و متخصص جنگو هستی.

# زمینه:
من یک اپ جنگو دارم که فایل مستند کد آن (`[NAME]_doc.md`) را که توسط اسکریپت `docgen.sh` تولید شده است، در اختیار تو قرار می‌دهم.

# وظیفه تو:
بر اساس محتوای فایل ضمیمه شده، سه مستند زیر را برای من تولید کن:

## مستند اول: `ARCHITECTURE.md`
- نمودار C4 سطح ۳ (Component Diagram) با فرمت Mermaid
- شرح اجزاء (Module Dictionary) به صورت جدول شامل: نام، فایل، نوع، ورودی، خروجی، وابسته‌ها
- جریان داده (Data Flow) به صورت خطی (حداقل ۲ جریان: اصلی و ثانویه)
- نقشه دیباگ (کجا برای چه مشکلی شروع کنم)
- نقشه توسعه (برای افزودن ویژگی جدید از کجا شروع کنم)

## مستند دوم: `CODE_REVIEW.md` (بخش Dead Code)
- روش شناسایی دستی فانکشن‌های بی‌استفاده (۳ مرحله: جستجوی ایمپورت، جستجوی caller، چک‌لیست حذف ایمن)
- جدول فانکشن‌های مشکوک (با ستون‌های: فایل، فانکشن، آخرین استفاده، وضعیت، اقدام پیشنهادی)
- لیست فانکشن‌هایی که به نظر می‌رسد در هیچ جای دیگری استفاده نشده‌اند (بر اساس تحلیل ایمپورت‌ها)

## مستند سوم: `CODE_REVIEW.md` (بخش کیفیت ماژول‌ها)
- ماتریس ارزیابی برای هر فایل مهم بر اساس ۵ معیار (امنیت، عملکرد، تکراری، SOLID، Design Pattern) با امتیاز ۱ تا ۵
- جدول ارزیابی فایل‌ها با ستون‌های: فایل، امنیت، عملکرد، تکراری، SOLID، الگو، امتیاز کل، اقدام پیشنهادی
- شناسایی حداقل ۳ مشکل خاص واقعی از کد (با ذکر نام فایل و شماره خط تقریبی)
- ارائه راهکار برای هر مشکل
- اولویت‌بندی اقدامات (بالا/متوسط/پایین) با زمان تخمینی

# قالب خروجی:
- از زبان Markdown استفاده کن
- از آیکون‌های مناسب استفاده کن (📐, 📚, 🔄, 🐛, 🚀, 💀, 📊, 🔴, 🟡, 🟢)
- جداول باید خوانا و دارای هدر مشخص باشند
- کدهای Mermaid باید مستقیماً در Markdown قابل رندر باشند
- در بخش مشکلات واقعی، مستقیماً به کد ضمیمه شده ارجاع بده

# فایل ضمیمه:
[NAME]_doc.md

# توجه:
- اگر فانکشن یا کلاسی در کد هست اما هیچ ارجاعی به آن نمی‌بینی، در بخش Dead Code ثبت کن
- اگر الگوی طراحی خاصی تشخیص دادی (مانند Strategy, Factory, Singleton, Observer) در بخش کیفیت ذکر کن
- اگر هیچ مشکل امنیتی یا عملکردی واضحی نمی‌بینی، بنویس «مشکل واضحی دیده نمی‌شود»
- ابتدا مستند اراسل شده را عمیق و چند لایه تحلیل کن 
- در هر پاسخ فقط یکی از مستندهای خواست هشده را بده و با تایید من شروع کن به تولید مستند بعدی

# قوانین ترسیم دیاگرام Mermaid برای معماری اپ جنگو

## قوانین سینتکسی (بدون خطا)
- در متن لینک‌ها (متن بین `|...|`) از هیچ یک از کاراکترهای زیر استفاده نکن:
  - `(` یا `)` (پرانتز)
  - `/` (اسلش)
  - `_` (آندرلاین)
  - `#` (هش)
  - `?` (علامت سوال)
  - `=` (مساوی)
  - `<` یا `>` (بزرگتر/کوچکتر)
- در عوض، از فاصله یا خط تیره استفاده کن: مثلاً `POST form` نه `POST(form)`
- نام گره‌ها (مثلاً `AdminView["text"]`) می‌توانند پرانتز و اسلش داشته باشند، فقط متن لینک‌ها نمی‌توانند.

## قوانین ساختاری (بدون شلوغی)
- هر نمودار حداکثر ۱۲-۱۵ گره داشته باشد
- اگر معماری پیچیده است، به جای یک نمودار، ۲ یا ۳ نمودار تخصصی بساز:
  - نمودار ۱: نمای کلی (ورودی‌ها + لایه‌ها)
  - نمودار ۲: جزئیات سرویس‌ها و کش
  - نمودار ۳: جریان داده و قالب‌ها
- از `subgraph` فقط برای گروه‌بندی‌های سطح بالا استفاده کن
- نام `subgraph` باید ساده و بدون فاصله یا کاراکتر خاص باشد (مثلاً `Services` نه `Service Layer`)

## قوانین خوانایی
- از شکل‌های متفاوت استفاده کن:
  - `[("متن")]` برای دیتابیس‌ها و منابع داده
  - `["متن"]` برای کامپوننت‌ها و سرویس‌ها
  - `("متن")` برای بازیگران (ادمین، کاربر)
- از آیکون‌های یونیکد ساده (مثل 👤, 📡, 📁) در متن گره‌ها استفاده کن
- لینک‌ها را مختصر و مفید بنویس (حداکثر ۳ کلمه)

## مثال درست (✅) و غلط (❌)

❌ غلط:
``
Admin -->|POST(form)| AdminView
AdminView -->|get_data()| Service
Service -->|SELECT/UPDATE| DB
``

✅ درست:
``
Admin -->|POST form| AdminView
AdminView -->|get data| Service
Service -->|select update| DB
``

## الگوی آماده (برای اپ‌های جنگو)

نمودار ۱ (نمای کلی):
``mermaid
graph TD
    Admin[("👤 ادمین")]
    API[("📡 API Client")]

    Admin -->|GET POST| ViewAdmin["admin view"]
    API -->|GET POST| ViewAPI["api view"]

    ViewAdmin -->|calls| ServiceLayer["service layer"]
    ViewAPI -->|calls| ServiceLayer

    ServiceLayer -->|queries| Database[("database")]
    ServiceLayer -.->|cache| CacheService["cache service"]

    ViewAdmin -->|renders| Template["template"]
``

نمودار ۲ (جزئیات سرویس‌ها):
``mermaid
graph TD
    subgraph Services ["service layer"]
        Base["base service"]
        Svc1["service one"]
        Svc2["service two"]
    end

    Cache["cache service"]
    Signals["signals"]

    Svc1 -.->|inherits| Base
    Svc2 -.->|inherits| Base

    Svc1 -->|refresh| Cache
    Svc2 -->|refresh| Cache

    Signals -->|trigger| Cache
``

نمودار ۳ (قالب‌ها):
``mermaid
graph TD
    View["admin view"]
    Template["dashboard template"]
    Tab1["tab one"]
    Tab2["tab two"]

    View -->|renders| Template
    Template -->|includes| Tab1
    Template -->|includes| Tab2
``

## نکات نهایی
- قبل از ارسال نمودار، از نظر خطاهای سینتکسی بررسی کن
- اگر نمودار به هر دلیل خطا داد، آن را به ۲ نمودار کوچکتر تقسیم کن
- در صورت نیاز به نمایش وابستگی‌های پیچیده، از جدول شرح اجزاء (Module Dictionary) در کنار نمودار استفاده کن
``


# شروع کن.
```

---

## ویژگی‌های این پرامپت

| ویژگی | توضیح |
|:---|:---|
| **قابل تکرار** | هر بار با تغییر `[NAME]` و فایل ضمیمه، کار می‌کند |
| **بدون مذاکره** | همه توافقات قبلی در پرامپت گنجانده شده |
| **خروجی ساختاریافته** | سه مستند مجزا با قالب مشخص |
| **تحلیل واقعی** | از کد واقعی برای شناسایی مشکلات استفاده می‌کند |
| **قابل استفاده در هر هوش مصنوعی** | با Claude، GPT-4، Gemini، و غیره کار می‌کند |

---

## پیشنهاد تکمیلی (بهینه‌سازی برای تیم)

اگر می‌خواهی این فرآیند را **کاملاً خودکار** کنی، می‌توانی یک اسکریپت `archgen.sh` بنویسی که:

1. `docgen.sh` را اجرا کند
2. محتوای `[NAME]_doc.md` را بخواند
3. همان پرامپت بالا را به همراه فایل به API هوش مصنوعی ارسال کند
4. خروجی را به صورت `ARCHITECTURE.md` و `CODE_REVIEW.md` ذخیره کند

