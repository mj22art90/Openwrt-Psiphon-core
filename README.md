**پارسی** | [English](README.en.md)



حق با شماست؛ برای اینکه پروژه‌ی گیت‌هاب شما کاملاً آماده، یکپارچه و بدون هیچ نقص یا بخش جاافتاده‌ای باشد، تمام کدهای فایل **LUCI Final Psiphon** را همراه با تمام گزینه‌ها (مانند نوع پروتکل Transport، پورت‌های SOCKS و HTTP، دکمه‌ها و سیستم مانیتورینگ وضعیت آی‌پ‌ی) مستقیماً درون ساختار راهنمای جامع **OpenWrt 25** ادغام کردم.

در این نسخه، تمام دستورات بر پایه مدیریت بسته جدید **`apk`** بازنویسی شده و فایل سرویس (`init.d`) به گونه‌ای توسعه یافته که تمامی گزینه‌ها را به صورت پویا از پنل گرافیکی لوسی دریافت و اعمال کند.

متن زیر سند کامل و نهایی است که می‌توانید آن را به عنوان فایل اصلی راهنما (مثلاً `README.md`) در مخزن گیت‌هاب خود قرار دهید:

---

# راهنمای جامع نصب، راه‌اندازی و خودکارسازی Psiphon-Core به همراه پنل گرافیکی LuCI در OpenWrt 25

این پروژه یک راهنمای کاملاً بومی و عملیاتی برای کامپایل، کانفیگ و اتصال هسته لینوکسی سایفون (`psiphon-core`) به رابط کاربری گرافیکی لوسی (**LuCI JavaScript**) در سیستم‌عامل **OpenWrt 25** است. تمامی کلیدهای کنترل سرویس، فیلدهای تنظیمات (پورت‌ها، کشور، پروتکل) و بخش مانیتورینگ وضعیت آی‌پی کاملاً همگام‌سازی شده‌اند.

---

## 🛠️ ۱. آموزش کامپایل فایل باینری (روی کامپیوتر) - PowerShell

برای ساخت فایل اجرایی اختصاصی روتر خود، ابتدا مطمئن شوید زبان Go روی سیستم شما نصب است. سپس ترمینال را باز کرده و بر اساس معماری پردازنده روتر خود، دستورات زیر را اجرا کنید:

```powershell
# دریافت سورس کد رسمی هسته سایفون از مخزن گیت‌هاب
git clone https://github.com/Psiphon-Labs/psiphon-tunnel-core.git
cd psiphon-tunnel-core/ConsoleClient

# کامپایل برای روترهای ۶۴ بیتی (Aarch64 / ARM64 مانند GL.iNet MT3000 / MT2500)
$env:GOOS="linux"
$env:GOARCH="arm64"
go build -ldflags="-s -w" -o psiphon-core main.go

# کامپایل برای روترهای ۳۲ بیتی (ARMv7 مانند Google Wifi AC-1304)
$env:GOOS="linux"
$env:GOARCH="arm"
$env:GOARM="7"
go build -ldflags="-s -w" -o psiphon-core main.go

```

*پس از اتمام کامپایل، فایل خروجی `psiphon-core` را از طریق ابزارهایی مانند MobaXterm یا SCP به مسیر `/usr/bin/` روی روتر منتقل کنید.*

---

## 🚀 ۲. آماده‌سازی و نصب پیش‌نیازها در OpenWrt 25

در نسخه OpenWrt 25 ابزار قدیمی `opkg` حذف شده و سیستم از **`apk`** استفاده می‌کند. با اجرای دستور زیر، مجوزهای فایل باینری را صادر کرده و ابزارهای مانیتورینگ و شبکه را نصب کنید:

```bash
# صدور مجوز اجرا و ساخت پوشه دیتابیس سایفون
chmod +x /usr/bin/psiphon-core
mkdir -p /usr/bin/psiphon_data

# نصب پیش‌نیازهای شبکه و ابزار مانیتورینگ آی‌پی (مخصوص OpenWrt 25)
apk -U add socat curl wget-ssl coreutils-nohup

```

---

## 📁 ۳. استقرار زیرساخت و کدهای کامل پنل گرافیکی LuCI

بلوک کد زیر یک اسکریپت همه‌کاره است. آن را به طور کامل کپی کرده و در ترمینال روتر پیست کنید. این اسکریپت تمام فایل‌های ساختاری لوسی، تنظیمات UCI، کدهای جاوااسکریپت داشبورد (همراه با دکمه‌ها و فیلدهای کامل) و مجوزهای امنیتی ACL را به صورت یکجا ایجاد می‌کند:


```bash
cat << 'EOF' > /tmp/install_psiphon_luci.sh
#!/bin/sh

# ۱. ساخت پوشه‌های مورد نیاز سیستم
mkdir -p /www/luci-static/resources/view/services
mkdir -p /usr/share/luci/menu.d
mkdir -p /usr/share/rpcd/acl.d
mkdir -p /etc/config

# ۲. ایجاد فایل تنظیمات پیش‌فرض
cat << 'EOC' > /etc/config/psiphon
config psiphon 'config'
	option enabled '0'
	option country ''
	option transport 'STANDARD'
	option socks_port '10808'
	option http_port '10809'
EOC

# ۳. ساخت فایل جاوااسکریپت اصلی پنل لوسی (کد کامل داشبورد گرافیکی)
cat << 'EOC' > /www/luci-static/resources/view/services/psiphon.js
'use strict';
'required view';
'required form';
'required fs';
'required ui';
'required uci';
'required poll';

return L.view.extend({
	router_ip: '192.168.1.1',

	load: function() {
		return L.uci.load('network').then(L.bind(function() {
			var ip = L.uci.get('network', 'lan', 'ipaddr');
			if (ip) {
				this.router_ip = ip;
			}
			return L.uci.load('psiphon');
		}, this));
	},

	render: function(data) {
		var m, s, o;

		m = new L.form.Map('psiphon', _('Psiphon VPN Configuration'), 
			_('Unified Single-Page Control Panel for Psiphon Tunnel Core.'));

		s = m.section(L.form.NamedSection, 'config', 'psiphon', _('Service Status & Configuration'));
		s.anonymous = true;

		// چک‌مارک فعال‌سازی
		o = s.option(L.form.Flag, 'enabled', _('Enable'));
		o.rmempty = false;

		// انتخاب کشور با منوی کشویی کامل
		o = s.option(L.form.ListValue, 'country', _('Region'));
		o.value('', _('Best Performance'));
		o.value('AT', _('Austria'));
		o.value('BE', _('Belgium'));
		o.value('CA', _('Canada'));
		o.value('CH', _('Switzerland'));
		o.value('DE', _('Germany'));
		o.value('DK', _('Denmark'));
		o.value('ES', _('Spain'));
		o.value('FI', _('Finland'));
		o.value('FR', _('France'));
		o.value('GB', _('United Kingdom'));
		o.value('IT', _('Italy'));
		o.value('JP', _('Japan'));
		o.value('NL', _('Netherlands'));
		o.value('NO', _('Norway'));
		o.value('PL', _('Poland'));
		o.value('SE', _('Sweden'));
		o.value('SG', _('Singapore'));
		o.value('US', _('United States'));
		o.default = '';

		// انتخاب نوع پروتکل ارتباطی (Transport Mode)
		o = s.option(L.form.ListValue, 'transport', _('Transport Mode'));
		o.value('STANDARD', _('Standard'));
		o.value('QUIC', _('QUIC'));
		o.value('SSH', _('SSH'));
		o.default = 'STANDARD';

		// پورت SOCKS5 داخلی
		o = s.option(L.form.Value, 'socks_port', _('SOCKS Port'));
		o.datatype = 'port';
		o.default = '10808';

		// پورت HTTP داخلی
		o = s.option(L.form.Value, 'http_port', _('HTTP Port'));
		o.datatype = 'port';
		o.default = '10809';

		// بخش وضعیت آی‌پی و اتصالات
		var statusSection = m.section(L.form.NamedSection, 'config', 'psiphon', _('Live Connection Status'));
		statusSection.render = L.bind(function() {
			return E('div', { 'class': 'cbi-section' }, [
				E('div', { 'style': 'padding: 15px; background: #1a1c20; border: 1px solid #333; border-radius: 4px; color: #ccc;' }, [
					E('p', { 'style': 'margin: 5px 0;' }, [E('strong', { 'style': 'color: #88a;' }, '🔴 Real IP: '), E('span', { 'id': 'ip_real' }, '⏳ Checking...')]),
					E('p', { 'style': 'margin: 5px 0;' }, [E('strong', { 'style': 'color: #88a;' }, '🟢 Psiphon IP: '), E('span', { 'id': 'ip_psiphon' }, '⏳ Checking...')]),
					E('hr', { 'style': 'border-color: #444; margin: 15px 0;' }),
					E('div', { 'style': 'margin-top: 10px;' }, [
						E('button', {
							'class': 'btn cbi-button cbi-button-apply',
							'click': function(ev) {
								ev.preventDefault();
								L.fs.exec('/etc/init.d/psiphon', ['start']).then(function() {
									ui.addNotification(null, E('p', _('Starting Psiphon service...')), 'info');
								});
							}
						}, _('Start')),
						' ',
						E('button', {
							'class': 'btn cbi-button cbi-button-remove',
							'click': function(ev) {
								ev.preventDefault();
								L.fs.exec('/etc/init.d/psiphon', ['stop']).then(function() {
									ui.addNotification(null, E('p', _('Stopping Psiphon service...')), 'info');
								});
							}
						}, _('Stop'))
					])
				]),
				E('h3', { 'style': 'margin-top: 20px;' }, _('Terminal / Logs')),
				E('textarea', {
					'id': 'psiphon_log',
					'readonly': 'readonly',
					'style': 'width: 100%; height: 260px; font-family: monospace; font-size: 12px; background: #111; color: #00ff66; padding: 10px; border-radius: 4px; border: 1px solid #222;'
				}, 'Standby...')
			]);
		}, this);

		// مکانیزم پولینگ (اطلاعات زنده) برای خواندن وضعیت آی‌پی و لاگ‌ها
		L.poll.add(function() {
			var sPort = L.uci.get('psiphon', 'config', 'socks_port') || '10808';
			var cmdReal = 'curl -sL -m 5 [https://api.ipify.org](https://api.ipify.org) || wget -qO- --timeout=5 [https://api.ipify.org](https://api.ipify.org) || echo "Network Error"';
			var cmdPsiphon = 'curl -sL -m 5 --socks5-hostname 127.0.0.1:' + sPort + ' [https://api.ipify.org](https://api.ipify.org) || echo "Disconnected"';

			// آپدیت وضعیت آی‌پی واقعی شبکه
			L.fs.exec('/bin/sh', ['-c', cmdReal]).then(function(res) {
				var el = document.getElementById('ip_real');
				if (el) el.textContent = res.stdout.trim() || 'Network Error';
			});

			// آپدیت وضعیت آی‌پی پروکسی سایفون
			L.fs.exec('/bin/sh', ['-c', cmdPsiphon]).then(function(res) {
				var el = document.getElementById('ip_psiphon');
				if (el) {
					var out = res.stdout.trim();
					el.textContent = out;
					el.style.color = (out === 'Disconnected') ? '#ff4040' : '#00ff66';
				}
			});

			// خواندن فایل لاگ سیستم
			L.fs.exec('/usr/bin/tail', ['-n', '20', '/tmp/psiphon.log']).then(function(res) {
				var el = document.getElementById('psiphon_log');
				if (el) el.value = res.stdout.trim() || 'Service stopped or log empty.';
			});
		}, 5);

		return m.render();
	}
});
EOC

# ۴. ایجاد فایل منو لوسی در بخش Services
cat << 'EOC' > /usr/share/luci/menu.d/luci-app-psiphon.json
{
	"admin/services/psiphon": {
		"title": "Psiphon",
		"action": {
			"type": "view",
			"path": "services/psiphon"
		},
		"depends": {
			"acl": [ "luci-app-psiphon" ]
		}
	}
}
EOC

# ۵. ایجاد فایل جامع دسترسی‌های امنیتی سیستم (ACL JSON)
cat << 'EOC' > /usr/share/rpcd/acl.d/luci-app-psiphon.json
{
	"luci-app-psiphon": {
		"description": "Grant access to Psiphon control and execution",
		"read": {
			"file": {
				"/tmp/psiphon.log": [ "read" ]
			},
			"uci": [ "psiphon", "network" ]
		},
		"write": {
			"file": {
				"/tmp/psiphon.log": [ "write" ],
				"/bin/sh": [ "exec" ],
				"/usr/bin/tail": [ "exec" ],
				"/etc/init.d/psiphon": [ "exec" ]
			},
			"uci": [ "psiphon" ]
		}
	}
}
EOC

echo "LuCI Panel files with FULL country list successfully prepared."
EOF

# اجرای اسکریپت آماده‌سازی ساختار لوسی
chmod +x /tmp/install_psiphon_luci.sh
/tmp/install_psiphon_luci.sh
rm /tmp/install_psiphon_luci.sh

---

```

## ⚙️ ۴. ایجاد اسکریپت سرویس هوشمند سیستم (`/etc/init.d/psiphon`)

این اسکریپت مغز متفکر اتصال بک‌اند است. هنگام استارت، تمامی فیلدهای تکمیل‌شده در پنل لوسی (مانند کشور انتخابی، پورت‌ها و نوع پروتکل) را دریافت کرده، فایل کانفیگ اصلی سایفون را در لحظه بازنویسی می‌کند و پروسه هدایت اتصالات به پورت‌های محلی شبکه LAN روتر را انجام می‌دهد:

```bash
cat << 'EOF' > /etc/init.d/psiphon
#!/bin/sh /etc/rc.common

START=99
USE_PROCD=1

start_service() {
    # ۱. خواندن پیکربندی‌های ذخیره شده از uci لوسی
    config_load psiphon
    local enabled country transport socks_port http_port
    
    config_get_bool enabled config enabled 0
    config_get country config country 'ALL'
    config_get transport config transport 'STANDARD'
    config_get socks_port config socks_port '10808'
    config_get http_port config http_port '10809'
    
    # اگر تیک گزینه Enable خاموش باشد، اجرای سرویس متوقف می‌شود
    if [ "$enabled" -eq 0 ]; then
        echo "Psiphon is disabled in LuCI configuration. Skipping start." >> /tmp/psiphon.log
        return 0
    fi

    # ۲. ساخت و بروزرسانی پویا و آنی فایل کانفیگ JSON سایفون بر اساس متغیرهای انتخابی لوسی
    cat << JSON > /usr/bin/psiphon.config
{
    "DataRootDirectory": "/usr/bin/psiphon_data",
    "EgressRegion": "$country",
    "TransportProtocols": ["$transport"]
}
JSON

    # ۳. راه‌اندازی و مدیریت پروسه هوشمند هسته با سیستم پروکد
    procd_open_instance "psiphon"
    procd_set_param command /bin/sh -c "
        killall -9 psiphon-core socat 2>/dev/null
        
        # اجرای فایل باینری سایفون با کانفیگ بازنویسی شده
        /usr/bin/psiphon-core -config /usr/bin/psiphon.config > /tmp/psiphon.log 2>&1 &
        PSIPHON_PID=\$!
        
        # انتظار منطقی برای انجام موفق هندشیک‌ها و باز شدن پورت کلاینت
        echo 'Waiting for Psiphon to establish tunnels...' >> /tmp/psiphon.log
        while ! grep -q 'ListeningSocksProxyPort' /tmp/psiphon.log; do
            sleep 1
        done
        
        # استخراج اتوماتیک پورت‌های داینامیک داخلی ایجاد شده توسط هسته سایفون
        CORE_SOCKS=\$(grep 'ListeningSocksProxyPort' /tmp/psiphon.log | grep -o '\"port\":[0-9]*' | cut -d':' -f2)
        CORE_HTTP=\$(grep 'ListeningHttpProxyPort' /tmp/psiphon.log | grep -o '\"port\":[0-9]*' | cut -d':' -f2)
        
        # واکشی آی‌پی محلی فعلی روتر (LAN IP) برای پل زدن سراسری اتصالات
        ROUTER_IP=\$(ubus call network.interface.lan status | jsonfilter -e '@[\"ipv4-address\"][0].address')
        [ -z \"\$ROUTER_IP\" ] && ROUTER_IP='192.168.1.1'
        
        # اتصال پورت‌های لوکال هاست سایفون به پورت‌های ثابت تعریف شده در لوسی بر روی کل شبکه LAN
        socat TCP-LISTEN:\$socks_port,fork,bind=\$ROUTER_IP TCP:127.0.0.1:\$CORE_SOCKS &
        socat TCP-LISTEN:\$http_port,fork,bind=\$ROUTER_IP TCP:127.0.0.1:\$CORE_HTTP &
        
        echo \"Psiphon ports successfully bound to LAN IP \$ROUTER_IP on ports \$socks_port and \$http_port\" >> /tmp/psiphon.log
        
        wait \$PSIPHON_PID
    "
    procd_set_param respawn
    procd_close_instance
}

stop_service() {
    killall -9 psiphon-core socat 2>/dev/null || true
    echo "Psiphon core and socat tunnels stopped safely." > /tmp/psiphon.log
}
EOF

# اعطای دسترسی‌های اجرایی سرویس سیستم
chmod +x /etc/init.d/psiphon
/etc/init.d/psiphon enable

```

---

## 🔄 ۵. بارگذاری نهایی و اعمال تغییرات پنل گرافیکی

برای پاک کردن سیستم کش قدیمی رابط کاربری لوسی و بارگذاری کامل اسکریپت‌های امنیتی سیستم RPC، دستورات زیر را اجرا کنید:

```bash
chmod 644 /usr/share/rpcd/acl.d/luci-app-psiphon.json
chmod 644 /usr/share/luci/menu.d/luci-app-psiphon.json

# راه‌اندازی مجدد سرویس سیستم تبادل داده لوسی
/etc/init.d/rpcd restart

# حذف کامل کش‌های قدیمی برای ظاهر شدن آنی منوی Psiphon در تب Services
rm -rf /tmp/luci-indexcache /tmp/luci-modulecache

```

*حالا مرورگر خود را با کلیدهای ترکیبی `Ctrl + F5` در سیستم ریفرش کنید تا منوی سرویس با عملکرد کامل کلیدها بالا بیاید.*

---

## 🛡️ ۶. تنظیمات فایروال (مبتنی بر Nftables در OpenWrt 25)

برای اینکه پورت‌های جدید پروکسی روتر اجازه تبادل اطلاعات با سایر دستگاه‌های متصل به وای‌فای یا شبکه LAN را داشته باشند، پورت‌ها را در لایه فایروال محلی باز کنید:

```bash
# باز کردن پورت‌های ورودی فایروال شبکه برای اتصالات کلاینت‌ها
nft add rule inet fw4 input iifname "br-lan" tcp dport 10808-10809 accept 2>/dev/null || true

```

---

## 🩺 ۷. تست و عیب‌یابی شبکه از طریق ترمینال

پس از زدن دکمه **Start Service** در پنل لوسی، با دستورات زیر می‌توانید عملکرد فیلترشکن را به طور مستقیم در لایه شبکه روتر ارزیابی کنید:

* **تست عبور موفق ترافیک از پورت HTTP Proxy:**

```bash
curl -x http://127.0.0.1:10809 https://api.ipify.org

```

* **تست عبور موفق ترافیک از پورت SOCKS5 Proxy:**

```bash
curl --socks5-hostname 127.0.0.1:10808 https://api.ipify.org

```

---

## 🗑️ ۸. حذف کامل و بی‌بازگشت سایفون از سیستم (Uninstall)

اگر به هر دلیلی تمایل داشتید تمامی تنظیمات، فایل‌های باینری، دیتابیس‌ها و منوهای پنل لوسی سایفون را بدون به جا ماندن هیچ ردپایی حذف کنید، اسکریپت یکپارچه زیر را در ترمینال روتر اجرا کنید:

```bash
# ۱. متوقف کردن پردازش‌ها و غیرفعال‌سازی سرویس سیستم
/etc/init.d/psiphon disable 2>/dev/null
/etc/init.d/psiphon stop 2>/dev/null
killall -9 psiphon-core socat 2>/dev/null

# ۲. پاکسازی کامل فایل‌های سیستمی و پنل گرافیکی لوسی
rm -f /usr/bin/psiphon-core
rm -rf /usr/bin/psiphon_data
rm -f /usr/bin/psiphon.config
rm -f /etc/init.d/psiphon
rm -f /etc/config/psiphon
rm -f /www/luci-static/resources/view/services/psiphon.js
rm -f /usr/share/luci/menu.d/luci-app-psiphon.json
rm -f /usr/share/rpcd/acl.d/luci-app-psiphon.json
rm -f /tmp/psiphon.log

# ۳. راه‌اندازی مجدد بخش دسترسی‌ها و پاک کردن کامل سیستم کش لوسی
/etc/init.d/rpcd restart
rm -rf /tmp/luci-indexcache /tmp/luci-modulecache

echo "Psiphon app and all associated LuCI components successfully uninstalled."

```
