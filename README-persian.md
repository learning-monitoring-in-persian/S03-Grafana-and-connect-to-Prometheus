[فارسی](README-persian.md) | [English](README.md)

# راه‌اندازی Grafana

> ### نکته
>
> اگر قصد دارید **Grafana** را کنار **Prometheus** یا ابزارهای دیگر نصب کنید، شدیداً پیشنهاد می‌کنم از نسخه‌ی **Docker** استفاده کنید. این کار باعث می‌شود تنظیمات (مثل Provision کردن داشبوردها و دیتاسورس‌ها) و نگهداری (Maintenance) در آینده بسیار ساده‌تر شود.
>
> اما اگر ماشین شما داکر ندارد، می‌توانید نسخه‌ی باینری گرافانا را مستقیم نصب کنید.

## نصب Grafana روی لینوکس (Debian/Ubuntu)

گرافانا دو نسخه اصلی دارد: **Enterprise** (که شامل ویژگی‌های پولی است اما برای کارهای بیسیک رایگان است) و **OSS** (کاملاً رایگان و متن‌باز). ما در اینجا از نسخه‌ی **OSS** استفاده می‌کنیم.

بهترین و توصیه‌شده‌ترین روش نصب برای توزیع‌های دبیان/اوبونتو استفاده از ریپازیتوری رسمی (APT) است تا نیازی به دانلود باینری‌ها یا هاردکد کردن ورژن‌ها نباشد و آپدیت آن در آینده راحت‌تر باشد.

```bash
# ۱. پیش‌نیازها را نصب کنید
sudo apt-get install -y apt-transport-https software-properties-common wget gnupg

# ۲. کلید GPG گرافانا را اضافه کنید
sudo mkdir -p /etc/apt/keyrings
sudo wget -O /etc/apt/keyrings/grafana.asc https://apt.grafana.com/gpg-full.key
sudo chmod 644 /etc/apt/keyrings/grafana.asc

# ۳. ریپازیتوری گرافانا (نسخه Stable) را به لیست سورس‌ها اضافه کنید
echo "deb [signed-by=/etc/apt/keyrings/grafana.asc] https://apt.grafana.com stable main" | sudo tee -a /etc/apt/sources.list.d/grafana.list

# ۴. لیست پکیج‌ها را آپدیت کرده و گرافانا OSS را نصب کنید
sudo apt-get update
sudo apt-get install -y grafana
```

وقتی گرافانا را از طریق APT نصب می‌کنید، خودش به صورت خودکار کاربر، دایرکتوری‌ها و سرویس `systemd` را می‌سازد و نیازی به کارهای دستی نیست.

برای فعال‌سازی و اجرای سرویس، دستورات زیر را وارد کنید:

```bash
sudo systemctl daemon-reload
sudo systemctl enable grafana-server
sudo systemctl start grafana-server
```

> ### نکته
>
> اولین باری که گرافانا رو اجرا می‌کنید، ممکنه یک الی دو دقیقه طول بکشه تا دیتابیس اولیه ساخته بشه و پلاگین‌های پیش‌فرض دانلود بشن. تو این مدت وب‌سرور بالا نمی‌آید. می‌تونید با اجرای دستور `ss -ntlup | grep 3000` بررسی کنید که آیا پورت **3000** در حالت Listen قرار گرفته یا نه.

بعد از اجرا، Grafana روی پورت **3000** در دسترس است. برای دسترسی به پنل وب:

- `http://{IP_ADDRESS}:3000`

> ### نکته مهم
>
> مطمئن بشید که پورت `3000/tcp` توی فایروال ماشین شما (`ufw`, `iptables` و...) بازه تا بتونید به پنل وب دسترسی داشته باشید.

> ### نکته
>
> اطلاعات ورود (Login) پیش‌فرض گرافانا به این شکله:
> - **نام کاربری**: `admin`
> - **رمز عبور**: `admin`
>
> توی اولین ورود از شما خواسته میشه که این رمز رو عوض کنید.

## راه‌اندازی Grafana با Docker Compose

یک فایل `docker-compose.yml` به شکل زیر بسازید:

```yaml
services:
  grafana:
    image: grafana/grafana-oss:latest
    container_name: grafana
    restart: unless-stopped
    ports:
      - "3000:3000"
    environment:
      - GF_SECURITY_ADMIN_USER=admin
      - GF_SECURITY_ADMIN_PASSWORD=secure_password
    volumes:
      - grafana_data:/var/lib/grafana
      - ./grafana/provisioning:/etc/grafana/provisioning:ro
      - ./grafana/dashboards:/var/lib/grafana/dashboards:ro

volumes:
  grafana_data:
```

> ### نکته
>
> توی این کانفیگ داکر، ما از متغیرهای محیطی (`GF_SECURITY_ADMIN_USER` و `GF_SECURITY_ADMIN_PASSWORD`) برای تنظیم یوزر و پسورد ادمین استفاده کردیم. اگه این‌ها رو تنظیم نکنید، گرافانا به طور پیش‌فرض از همون `admin/admin` استفاده می‌کنه.

### ساختار پیشنهادی پوشه‌ها برای داکر

```text
.
├─ docker-compose.yml
└─ grafana/
   ├─ provisioning/
   │  ├─ datasources/
   │  │  └─ prometheus.yml
   │  └─ dashboards/
   │     └─ dashboards.yml
   └─ dashboards/
      └─ node_exporter.json     #1860_rev45.json
```

سپس کانتینر را اجرا کنید:

```bash
docker compose up -d
```

## اتصال Grafana به Prometheus

گرافانا برای نمایش داده‌ها باید بدونه Prometheus کجاست. دو راه برای این کار وجود داره:

### روش اول: از طریق پنل گرافانا (دستی)
۱. به **Connections -> Data sources** (یا **Add new connection**) برید.
۲. روی **Add data source** کلیک کنید و **Prometheus** رو انتخاب کنید.
۳. تو بخش URL، آدرس پرومتئوس رو وارد کنید (مثلاً `http://{PROMETHEUS_IP}:9090`).
۴. روی **Save & test** کلیک کنید.

### روش دوم: از طریق فایل کانفیگ Provisioning (پیشنهادی برای اتوماسیون)
شما می‌توانید گرافانا را طوری کانفیگ کنید که خودش به صورت خودکار پرومتئوس را بشناسد.

فایل `grafana/provisioning/datasources/prometheus.yml` را بسازید (در حالت داکر این فایل به دایرکتوری `/etc/grafana/provisioning/datasources/` مَپ می‌شود):

```yaml
apiVersion: 1

datasources:
  - name: Prometheus
    type: prometheus
    access: proxy
    url: http://{PROMETHEUS_IP}:9090 # آدرس پرومتئوس خود را جایگزین کنید
    isDefault: true
```

> ### نکته
>
> برای توضیحات کامل‌تر می‌توانید [داکیومنت Data source provisioning](https://grafana.com/docs/grafana/latest/administration/provisioning/#data-sources) گرافانا را مطالعه کنید.

## اضافه کردن داشبورد (مثال: Node Exporter)

شما می‌توانید در گرافانا داشبوردهای اختصاصی خودتان را بسازید یا از داشبوردهای آماده جامعه کاربری (Community) استفاده کنید. برای Node Exporter، دو تا از معروف‌ترین داشبوردها این موارد هستند:
- [Node Exporter Full (1860)](https://grafana.com/grafana/dashboards/1860-node-exporter-full/)
- [Node Exporter Dashboard (24784)](https://grafana.com/grafana/dashboards/24784-node-exporter-dashboard-20240520/)

برای اضافه کردن داشبورد دو راه دارید:

### روش اول: از طریق پنل گرافانا (دستی)
۱. در منو به مسیر **Dashboards -> New -> Import** بروید.
۲. **با استفاده از ID**: آیدی داشبورد (مثلاً `1860`) را وارد کرده و روی **Load** کلیک کنید.
۳. **با استفاده از JSON**: یا اینکه می‌توانید محتوای فایل JSON داشبورد را مستقیماً در باکس متنی کپی/پیست کنید.
۴. دیتاسورس Prometheus خود را انتخاب کرده و روی **Import** کلیک کنید.

### روش دوم: از طریق فایل کانفیگ Provisioning (پیشنهادی)
می‌توانید فایل JSON داشبورد را در سرور قرار دهید تا گرافانا هنگام بالا آمدن خودش آن را لود کند.

۱. **دانلود فایل JSON**: وارد لینک داشبورد در سایت گرافانا شوید (مثلاً داشبورد `1860`)، فایل JSON آن را دانلود کنید و در مسیر `grafana/dashboards/node_exporter.json` قرار دهید (این مسیر در داکر به `/var/lib/grafana/dashboards` و در باینری به شکل دستی به `/var/lib/grafana/dashboards` مپ می‌شود).
۲. **تنظیم کانفیگ Provisioning**: فایل `grafana/provisioning/dashboards/dashboards.yml` را با محتوای زیر بسازید:

```yaml
apiVersion: 1

providers:
  - name: 'Default'
    orgId: 1
    folder: ''
    type: file
    disableDeletion: false
    updateIntervalSeconds: 10
    options:
      path: /var/lib/grafana/dashboards # مسیری که گرافانا در آن به دنبال فایل‌های جیسون می‌گردد
```

> ### نکته مهم
>
> وقتی داشبوردها را از طریق فایل JSON به صورت اتوماتیک (Provisioning) اضافه می‌کنید، ممکن است UID دیتاسورس داخل فایل JSON با دیتاسورس پرومتئوس شما یکی نباشد. اگر دیدید پنل‌های داشبورد ارور **Datasource not found** یا **No data** می‌دهند، باید فایل JSON را باز کنید و متغیرهای دیتاسورس (مثلاً `${DS_PROMETHEUS}`) را با نام یا UID دیتاسورس خودتان جایگزین کنید. یا اینکه در فایل `prometheus.yml` (بخش دیتاسورس‌ها) یک `uid` مشخص تعریف کنید که با فایل JSON داشبورد همخوانی داشته باشد.

> ### نکته
>
> برای جزئیات بیشتر به [داکیومنت Dashboard provisioning](https://grafana.com/docs/grafana/latest/administration/provisioning/#dashboards) گرافانا مراجعه کنید.

## قابلیت Explore

گرافانا در منوی کناری بخشی به نام **Explore** دارد.
این بخش به شما اجازه می‌دهد بدون نیاز به ساختن هیچ داشبوردی، مستقیماً کوئری بنویسید (مثل [PromQL](https://prometheus.io/docs/prometheus/latest/querying/basics/)) و متریک‌ها، لاگ‌ها یا Traceها را ببینید. این قابلیت برای پیدا کردن متریک‌ها و عیب‌یابی (Troubleshooting) به شدت کاربردی است.

## سیستم هشداردهی (Grafana Alerts)

گرافانا خودش یک سیستم **Alerting** قدرتمند و داخلی دارد (مشابه Alertmanager در پرومتئوس اما با پنل گرافیکی بسیار راحت‌تر).

جریان (Flow) هشداردهی در گرافانا به این شکل است:
۱. **Alert Rule (قانون هشدار)**: یک کوئری می‌نویسید (مثلاً "آیا مصرف CPU بالای ۸۰٪ است؟").
۲. **Contact Point (نقطه‌ی تماس)**: تعیین می‌کنید که هشدارها به کجا ارسال شوند. گرافانا از Endpointهای بسیار زیادی مثل **Slack, Email, Telegram, Discord, Webhook** پشتیبانی می‌کند.
۳. **Notification Policy (سیاست‌های اطلاع‌رسانی)**: تعیین می‌کند کدام هشدارها به کدام Contact Point بروند (مثلاً "هشدارهای Critical به تلگرام، هشدارهای Warning به ایمیل").

تمام این موارد را می‌توانید از طریق منوی **Alerting** به صورت گرافیکی تنظیم کنید.

> ### نکته حرفه‌ای (Alert Provisioning)
>
> دقیقاً مثل دیتاسورس‌ها و داشبوردها، شما می‌توانید **Alert**های گرافانا را هم از طریق فایل‌های YAML (به‌صورت Code) مدیریت کنید! کافیست فایل‌های کانفیگ الرت (شامل Ruleها، Contact Pointها و Notification Policyها) را در پوشه `grafana/provisioning/alerting/` قرار دهید (که به `/etc/grafana/provisioning/alerting/` مپ می‌شود) تا گرافانا آن‌ها را اتوماتیک لود کند. برای دیدن نمونه فایل‌ها به [داکیومنت Alerting Provisioning](https://grafana.com/docs/grafana/latest/administration/provisioning/#alerting) مراجعه کنید.

> ### نکته
>
> برای دیدن لیست کامل پلتفرم‌های پشتیبانی شده و نحوه تنظیم آن‌ها به [داکیومنت Grafana Alerting](https://grafana.com/docs/grafana/latest/alerting/) مراجعه کنید.

## کاربران، نقش‌ها و کنترل دسترسی (Users, Roles, RBAC)

اگر با گرافانا آشنایی ندارید، درک سیستم دسترسی آن بسیار ساده است:

- **کاربران (Users)**: اشخاصی هستند که وارد سیستم می‌شوند (مثل اصغر، اقدس).
- **نقش‌ها (Roles)**: نقش‌ها تعیین می‌کنند یک کاربر چه *کاری* می‌تواند انجام دهد. نقش‌های پایه (Basic Roles) شامل موارد زیر است:
  - **Admin**: ادمین کل است. می‌تواند کاربر جدید بسازد، دیتاسورس اضافه کند و تنظیمات کلی را عوض کند.
  - **Editor**: می‌تواند داشبورد بسازد و ویرایش کند، اما نمی‌تواند تنظیمات سیستمی (مثل افزودن دیتاسورس) را تغییر دهد.
  - **Viewer**: فقط و فقط می‌تواند داشبوردها را ببیند. به هیچ‌وجه نمی‌تواند چیزی را تغییر دهد یا خراب کند.
- **تیم‌ها (Teams/Groups)**: به جای اینکه به کاربران یکی یکی دسترسی دهید، می‌توانید آن‌ها را در یک "تیم" قرار دهید (مثلاً "تیم زیرساخت"، "تیم توسعه‌دهندگان") و به آن تیم اجازه دسترسی به پوشه‌ای خاص از داشبوردها را دهید.

تمام این مدیریت‌ها در بخش **Administration -> Users and access** انجام می‌شود.

> ### نکته
>
> برای ساختارهای پیچیده‌تر، گرافانا از سیستمی به نام Role-Based Access Control (RBAC) پشتیبانی می‌کند. برای اطلاعات بیشتر به [داکیومنت Grafana RBAC](https://grafana.com/docs/grafana/latest/administration/roles-and-permissions/) مراجعه کنید.

---

> ### پیشنهاد دوستانه
>
> گرافانا ابزار بسیار قدرتمند و بزرگی است. برای یادگیری بهتر و بیشتر، بهترین کار این است که در پنل گرافانا بگردید، وارد منوهای مختلف شوید و سعی کنید خودتان کوئری بزنید و پنل‌ها و داشبوردهای مختلف بسازید تا با قسمت‌های مختلف آن به خوبی آشنا شوید. موفق باشید! :)
