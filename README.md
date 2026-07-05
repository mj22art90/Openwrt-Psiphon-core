این هم کل دستورات و کدهای مورد نیاز، از مرحلهٔ کامپایل روی کامپیوتر گرفته تا کدهای مربوط به گیت‌هاب و اسکریپت نهایی روتر، همه در یک‌جا و به صورت دسته‌بندی‌شده:

### ۱. دستورات کامپایل فایل `psiphon-core` (روی کامپیوتر)

اگر می‌خواهید خودتان فایل باینری را برای پردازنده روتر (`aarch64_cortex-a53`) بسازید، ترمینال کامپیوتر را باز کرده و این خطوط را اجرا کنید:

```bash
# دریافت سورس کد و ورود به پوشه هسته
git clone https://github.com/Psiphon-Inc/psiphon-v3.git
cd psiphon-v3/psiphon-core

# دانلود وابستگی‌ها و تنظیم معماری لینوکس ۶۴ بیتی ARM
go mod download
export GOOS=linux
export GOARCH=arm64

# کامپایل بهینه شده و سبک برای روتر
go build -ldflags="-s -w" -o psiphon-core .

```

---

### ۲. دستور ساخت فایل `README.md` برای گیت‌هاب (روی کامپیوتر)

برای اینکه صفحه اول ریپازیتوری شما در گیت‌هاب کامل و شکیل باشد، این دستور را در پوشه پروژه روی کامپیوتر بزنید تا فایل مارک‌داون مستقیماً ساخته شود:

```bash
cat << 'EOF' > README.md
# OpenWrt Psiphon-Core Core Setup Guide

این پروژه راهنمایی جامع و اسکریپتی هوشمند برای راه‌اندازی و جفت‌کردن هسته لینوکسی سایفون (`psiphon-core`) با روترهای مبتنی بر **OpenWrt** و **GL.iNet** (مخصوصاً پردازنده‌های معماری `aarch64_cortex-a53`) است.

به دلیل محدودیت‌های دسترسی، سایفون در هر بار اجرا پورت‌های SOCKS و HTTP رندوم باز می‌کند. این اسکریپت به صورت داینامیک پورت‌ها را شکار کرده و روی پورت‌های ثابت شبکه داخلی لایو می‌کند.

---

## 🚀 گام اول: آماده‌سازی و انتقال فایل‌ها

۱. فایل باینری کامپایل شده `psiphon-core` را از طریق SFTP (برنامه‌هایی مثل MobaXterm یا WinSCP) به مسیر `/usr/bin/` روتر منتقل کنید.
۲. با دستور زیر دسترسی اجرای آن را فعال کنید:

```bash
chmod +x /usr/bin/psiphon-core

```

۳. پوشه دیتای داخلی سایفون را روی روتر ایجاد کنید:

```bash
mkdir -p /usr/bin/psiphon_data

```

---

## 🛠️ گام دوم: نصب ابزار پیش‌نیاز (`socat`)

ابزار `socat` وظیفه ریدایرکت ترافیک از پورت‌های رندوم داخلی به پورت‌های ثابت شبکه را بر عهده دارد. با دستور زیر آن را نصب کنید:

```bash
opkg update && opkg install socat

```

---

## 📝 گام سوم: ساخت فایل کانفیگ (`psiphon.config`)

دستور زیر را به صورت کامل کپی کرده و در ترمینال روتر پیست کنید تا فایل تنظیمات استاندارد و سبک در مسیر مشخص شده ساخته شود:

```bash
cat << 'EOF' > /usr/bin/psiphon.config
{
  "ClientPlatform": "Windows_10.0.26200_11",
  "ClientVersion": "187",
  "DataRootDirectory": "/usr/bin/psiphon_data",
  "EgressRegion": "US",
  "PropagationChannelId": "92AACC5BABE0944C",
  "SponsorId": "1BC527D3D09985CF",
  "ServerEntrySignaturePublicKey": "sHuUVTWaRyh5pZwy4UguSgkwmBe0EHtJJkoF5WrxmvA=",
  "UseIndistinguishableTLS": true
}
EOF

```

---

## ⚡ گام چهارم: اسکریپت راه‌انداز داینامیک و هوشمند

> ⚠️ **نکته بسیار مهم:** اگر آی‌پي داخلی روتر شما چیزی غیر از `192.168.18.1` است، در خطوط مربوط به `socat` (بخش ۵)، آی‌پي روتر خودتان را جایگزین کنید.

کد زیر را کپی کرده و در ترمینال روتر اجرا کنید تا سایفون استارت خورده و پورت‌های شبکه فعال شوند:

```bash
# ۱. بستن پروسه‌های قدیمی برای جلوگیری از تداخل پورت‌ها
killall -9 psiphon-core socat 2>/dev/null

# ۲. اجرای هسته اصلی سایفون در پس‌زمینه
/usr/bin/psiphon-core -config /usr/bin/psiphon.config -dataRootDirectory /usr/bin/psiphon_data > /tmp/psiphon.log 2>&1 &

# ۳. چند ثانیه صبر تا سایفون تانل را برقرار و پورت‌های داخلی را باز کند
echo "در حال راه‌اندازی سایفون... ۷ ثانیه صبر کنید..."
sleep 7

# ۴. استخراج خودکار پورت‌های رندوم سایفون از داخل لاگ فایل
SOCKS_PORT=\$(http_port=\$(cat /tmp/psiphon.log | grep -o '"port":[0-9]*' | cut -d':' -f2); echo \$http_port | awk '{print \$1}')
HTTP_PORT=\$(http_port=\$(cat /tmp/psiphon.log | grep -o '"port":[0-9]*' | cut -d':' -f2); echo \$http_port | awk '{print \$2}')

echo "پورت‌های شناسایی شده -> SOCKS: \$SOCKS_PORT | HTTP: \$HTTP_PORT"

# ۵. برقراری پل ارتباطی توسط socat به آی‌پي روتر (پورت‌های ثابت ۱۰۸۰۸ و ۱۰۸۰۹)
socat TCP-LISTEN:10809,fork,bind=192.168.18.1 TCP:127.0.0.1:\$HTTP_PORT &
socat TCP-LISTEN:10808,fork,bind=192.168.18.1 TCP:127.0.0.1:\$SOCKS_PORT &

# ۶. باز کردن فایروال روتر برای قبول دسترسی از شبکه داخلی
iptables -I INPUT -p tcp -i br-lan --dport 10808:10809 -j ACCEPT

echo "سایفون با موفقیت با پورت‌های ثابت روی شبکه داخلی جفت شد!"

```

---

## 💻 گام پنجم: تنظیمات کلاینت‌ها (ویندوز، گوشی، مرورگر)

پس از اجرای موفقیت‌آمیز مراحل بالا، برای استفاده از اینترنت بدون فیلتر کافیست مشخصات پروکسی زیر را در افزونه‌های مرورگر (مانند FoxyProxy) یا تنظیمات پروکسی سیستم‌عامل وارد کنید:

* **Proxy Type:** `HTTP`
* **IP Address:** `192.168.18.1` (آی‌پي روتر شما)
* **Port:** `10809`
EOF

```

---

### ۳. کدهای آماده‌سازی و کانفیگ نهایی (داخل ترمینال روتر جدید)

وقتی فایل `psiphon-core` را آپلود کردید، این دستورات را به ترتیب در ترمینال روتر بزنید تا پیش‌نیازها نصب و کانفیگ اعمال شود:

```bash
# مجاز کردن فایل باینری و ساخت پوشه دیتا
chmod +x /usr/bin/psiphon-core
mkdir -p /usr/bin/psiphon_data

# نصب ابزار socat روی روتر جدید
opkg update && opkg install socat

# ساخت خودکار فایل کانفیگ در روتر
cat << 'EOF' > /usr/bin/psiphon.config
{
  "ClientPlatform": "Windows_10.0.26200_11",
  "ClientVersion": "187",
  "DataRootDirectory": "/usr/bin/psiphon_data",
  "EgressRegion": "US",
  "PropagationChannelId": "92AACC5BABE0944C",
  "SponsorId": "1BC527D3D09985CF",
  "ServerEntrySignaturePublicKey": "sHuUVTWaRyh5pZwy4UguSgkwmBe0EHtJJkoF5WrxmvA=",
  "UseIndistinguishableTLS": true
}
EOF

```

---

### ۴. اسکریپت نهایی استارت و فعال‌سازی پل ارتباطی (داخل ترمینال روتر)

این همان اسکریپت هوشمندی است که تانل را باز کرده، پورت‌های داینامیک را شناسایی می‌کند و روی پورت‌های ثابت `10808` و `10809` تحویل کلاینت می‌دهد:

```bash
# ۱. بستن پروسه‌های قدیمی
killall -9 psiphon-core socat 2>/dev/null

# ۲. اجرای هسته اصلی سایفون در پس‌زمینه
/usr/bin/psiphon-core -config /usr/bin/psiphon.config -dataRootDirectory /usr/bin/psiphon_data > /tmp/psiphon.log 2>&1 &

# ۳. تایمر انتظار برای اتصال اولیه
echo "در حال راه‌اندازی سایفون... ۷ ثانیه صبر کنید..."
sleep 7

# ۴. استخراج پورت‌های متغیر
SOCKS_PORT=$(http_port=$(cat /tmp/psiphon.log | grep -o '"port":[0-9]*' | cut -d':' -f2); echo $http_port | awk '{print $1}')
HTTP_PORT=$(http_port=$(cat /tmp/psiphon.log | grep -o '"port":[0-9]*' | cut -d':' -f2); echo $http_port | awk '{print $2}')

echo "پورت‌های شناسایی شده -> SOCKS: $SOCKS_PORT | HTTP: $HTTP_PORT"

# ۵. هدایت پورت‌ها به شبکه داخلی روتر (در صورت تغییر آی‌پي روتر، آدرس زیر را اصلاح کنید)
socat TCP-LISTEN:10809,fork,bind=192.168.18.1 TCP:127.0.0.1:$HTTP_PORT &
socat TCP-LISTEN:10808,fork,bind=192.168.18.1 TCP:127.0.0.1:$SOCKS_PORT &

# ۶. باز کردن زنجیره فایروال برای پورت‌های ۱۰۸۰۸ و ۱۰۸۰۹
iptables -I INPUT -p tcp -i br-lan --dport 10808:10809 -j ACCEPT

echo "پل ارتباطی socat با موفقیت برقرار شد!"

```





