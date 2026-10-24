# FIKR. — فِكْر store
Stack: Node 22.13+, Express, SQLite (built-in node:sqlite, no native build), vanilla JS front end. No build step.

## تشغيل على جهازك (Local)
```
npm install
cp .env.example .env        # حط SESSION_SECRET (أي نص طويل)
npm run dev                 # http://localhost:3000   الأدمن: /admin.html
npm run create-admin -- you@example.com "a-long-password"
```
على جهازك الداتابيز والصور بتتحفظ جوه فولدر المشروع (`fikr.db` و`uploads/`).

## النشر على Railway (خطوة بخطوة)
1. ارفع المشروع على GitHub (ملف `.env` مش بيترفع، متأكد من `.gitignore`). نفّذ `npm install` مرة واحدة على جهازك وارفع `package-lock.json` معاه.
2. في Railway: **New Project → Deploy from GitHub repo**. الإعدادات جاهزة في `railway.json` (أمر التشغيل ومسار الفحص `/healthz`).
3. **مهم جداً، التخزين الدائم:** من الـ service اختار **Volumes → New Volume** وخلي الـ Mount path = `/data`. من غير الـ Volume الداتابيز والصور بتتمسح مع كل Deploy.
4. من تاب **Variables** ضيف:
   - `NODE_ENV=production`
   - `SESSION_SECRET` = نص عشوائي 32+ حرف (الموقع مش هيشتغل لو ناقص أو ضعيف)
   - `DATA_DIR=/data`
   - `ADMIN_EMAIL` و `ADMIN_PASSWORD` (10 حروف+). بيتعمل أدمن أول مرة بس، وبعدها امسح `ADMIN_PASSWORD`
   - `OWNER_EMAIL` و `OWNER_PHONES` و `TZ=Africa/Cairo`
   - الإيميل: انظر تحت (`BREVO_API_KEY` و `MAIL_FROM`)
5. **Settings → Networking → Generate Domain** (أو اربط دومين بتاعك)، وبعدها حط `SITE_URL` بالعنوان ده.
6. افتح `/healthz` لازم يرد `ok`، وبعدها `/admin.html` وسجل دخول، ومن Admin → alerts اضغط Send test.
7. خلي الـ service على **replica واحدة** (الـ Volume والـ SQLite مش بيشتغلوا مع أكتر من نسخة).

### الإيميل على Railway
Railway بتقفل بورتات SMTP (يعني Gmail مباشر) على الخطط Free وTrial وHobby، وبتفتحها على Pro بس. عشان كده الإيميل بيروح عن طريق API على HTTPS:
- **Brevo (الأنسب، من غير دومين):** سجل في brevo.com، ثم Senders → ضيف `fekrstore10@gmail.com` وفعّله، وبعدها SMTP & API → اعمل API key. حط `BREVO_API_KEY` و `MAIL_FROM=fekrstore10@gmail.com`. لو الرسائل راحت Spam في الأول، علّم عليها "Not spam" أو استخدم إيميل على دومين بتاعك.
- **Resend:** `RESEND_API_KEY` و `MAIL_FROM`، ومحتاج دومين متفعّل عشان تبعت لعملاء تانيين.
- **Gmail SMTP:** (`GMAIL_USER` + `GMAIL_APP_PASSWORD`) بيشتغل بس لو خطتك Pro أو على استضافة تانية.
طلبات الاسترجاع بتوصل على `OWNER_EMAIL` والصور مرفقة (لحد 15MB للإيميل، وباقي الصور موجودة في Admin → returns).

### ملاحظات التشغيل
- الداتابيز والجلسات (تسجيل الدخول) والصور كلها في `/data`، فمفيش حد بيتعمله logout عند كل Deploy.
- ممكن تاخد نسخة احتياطية بتحميل ملف `fikr.db` من الـ Volume بشكل دوري. راجع إعدادات الـ Volume في Railway لو فيها Backups على خطتك.
- كل Deploy بيحصل فيه ثواني توقف لأن الـ Volume بيتفصل من النسخة القديمة قبل ما يتوصل بالجديدة.
- الداتابيز بتتحدث لوحدها (جداول وأعمدة جديدة) من غير ما بياناتك القديمة تتأثر.

## Using it
Admin → Products → Add product: name, category, price, sale price, M/L/XL stock, upload images (first = main). Orders and settings (shipping zones, free-shipping threshold, cash on delivery, announcement) are in the other tabs. Cancelling an order restocks it.

## Assets
`public/assets/`: `logo-full*.png` / `logo-mark*.png` (transparent, light + dark; built from your logo JPG with `python3 scripts/make_logo.py logo.jpg`), `favicon.png`, `brain-light.svg`, `brain-dark.svg` (`python3 scripts/make_assets.py`). The PNG logos are raster: for perfectly sharp print or very large sizes, supply the original vector logo.

## Not included / to configure
Online payment gateway (only COD is real), srcset/WebP conversion, rate limiting beyond admin login.

## Customer accounts
Customers register/login at `#/login` (bcrypt hashes, server sessions). Logged-in customers get `member_discount`% off the subtotal automatically (default 5, change in Admin → Settings). Guests see a sign-up note in the bag, at checkout and on the login page; it disappears once they're logged in. Guest checkout still works. Order history at `#/account`.

## Notifications
On every new order the owner is alerted and the customer gets a confirmation. Failures never block an order; every attempt is logged in Admin → alerts (with a test button). Unconfigured channels are logged as "skipped". Free options first:
- **Telegram (owner, free, recommended):** in Telegram talk to @BotFather → `/newbot` → put the token in `TELEGRAM_BOT_TOKEN`. Each owner opens the bot and presses Start. In Admin → alerts press "Find Telegram chat IDs" and put the ids in `TELEGRAM_CHAT_IDS` (comma separated).
- **Email:** on Railway use Brevo or Resend (see above); Gmail app-password SMTP works on hosts that allow SMTP. Customers get an HTML email if they entered one.
- **SMS (free software, uses your SIM's SMS plan):** install the open-source SMSGate app (sms-gate.app) on an Android phone with an Egyptian SIM, turn on Cloud Server mode, and put the username/password it shows in `SMSGATE_USER` / `SMSGATE_PASSWORD`. The phone must stay on with internet.
- **WhatsApp to the owner (free, unofficial, personal use):** CallMeBot. Each owner follows the activation steps on callmebot.com (send "I allow callmebot to send me messages" to its number) and receives a key. Set `CALLMEBOT_KEYS=01154522332:KEY1,01021255949:KEY2`.
- **WhatsApp to customers:** the official Meta WhatsApp Cloud API is free to set up but business-initiated messages are billed per message (rates changed on 1 Oct 2026). Set `WHATSAPP_TOKEN`, `WHATSAPP_PHONE_ID` and an approved template in `WHATSAPP_TEMPLATE` (body variables: 1 first name, 2 order ref, 3 total) if you want it. Until then customers get email plus a free "Message us on WhatsApp" button on the confirmation page (`STORE_WHATSAPP` or the first owner phone).
- Owner numbers come from `OWNER_PHONES` (Egyptian `01…` is converted to `+20…`). Set `SITE_URL` to your https domain so the logo shows in emails.
- **One-tap WhatsApp confirmation (free):** each Telegram alert ends with a "send confirmation to the customer on WhatsApp" link, and every order in Admin → orders has a **Confirm on WhatsApp** button. Both open WhatsApp with the confirmation already written; you press send. No API, no cost.

## Photo flip, location, reports, category backgrounds
- **Flip:** products with 2+ photos show a brain button on the card and the product page; each press flips the photo to the next one (the order is the upload order; "Make main" in Admin changes which is first).
- **Live location:** checkout has "Share my live location" (browser GPS, asks permission, needs https in production, works on localhost). The order stores latitude/longitude; the owner alert includes a Google Maps link and Admin → orders has **Open map**.
- **Reports:** Admin → reports. Set each product's **Cost price (per piece)** in Products. Pick This week / Last week / Last 4 weeks / This month / All time / custom dates to see pieces, cost, sell price and profit per piece, revenue, cost and profit by category, by week (weeks start Saturday) and by product, plus period totals. Cost is saved on each order line when it is placed, so later cost changes don't rewrite old orders (orders placed before you set costs use the product's current cost).
- **Category backgrounds:** each category page has its own artwork (`public/assets/cat-*-light|dark.svg`, generated by `scripts/make_assets.py`): hoodies = a faceted hood, sweatpants = waistband and legs, zip-ups = an opening zipper, sets = two interlocking pieces.

## Crewnecks, sizes, colors, ticker, images
- **Crewnecks** is a 5th category (`crewnecks`) with its own shortcut, page and background artwork. **Sizes** are now M, L, XL, XXL, XXXL (XXL/XXXL only show on a product page when in stock).
- **Colors:** every color is its own product with its own page, photos and stock. In Admin → products, fill *Color name* + color, save, press **Add another color** (details are copied, you set the new color, photos and stock). Products in the same group show color swatches on each page and dots on their cards.
- **Announcement bar:** Admin → settings → *Announcement bar*; it scrolls across the top (hover to pause).
- **Photos:** Save the product, then add photos one by one in the edit screen. Each upload is saved immediately, "Make main" and "Remove" are always there.
- **SMS not arriving:** open Admin → alerts → *Send test*. `failed` shows the error (401 = wrong SMSGate username/password). `sent` with *Check delivery*: Pending = the phone did not pick it up, so open the SMSGate app, make sure Cloud Server is ON and online, allow SMS permission, turn off battery optimization for it and allow background activity/notifications, check the SIM has SMS credit (on dual SIM pick the SIM in the app), then test again.


## Railway: الإعداد الجاهز للنسخ (الإيميل + تليجرام 3 حسابات)
في Railway → Variables حط دول (القيم الفاضية بتاعتك أنت):
```
NODE_ENV=production
SESSION_SECRET=<نص عشوائي 32+ حرف>
DATA_DIR=/data
TZ=Africa/Cairo
SITE_URL=https://<دومين-الموقع>
OWNER_EMAIL=fekrstore10@gmail.com
OWNER_PHONES=01154522332,01021255949
MAIL_FROM=fekrstore10@gmail.com
BREVO_API_KEY=<من Brevo>
TELEGRAM_BOT_TOKEN=<توكن البوت>
TELEGRAM_CHAT_IDS=<id1>,<id2>,<id3>
```
- **الإيميل:** Brevo → Senders → ضيف `fekrstore10@gmail.com` وفعّله → SMTP & API → اعمل API key. الإيميلات (طلب جديد للأدمن + رسايل العميل) بتتبعت من العنوان ده.
- **تليجرام (3 حسابات):** كل واحد من التلاتة يفتح البوت ويدوس Start ويبعت أي رسالة. بعدها من الأدمن → alerts → *Find Telegram chat IDs* هتظهرلك الـ IDs، حطها في `TELEGRAM_CHAT_IDS` مفصولة بفاصلة. وبعدها *Send test alert*.
- `SITE_URL` لازم يكون https عشان لينكات واتساب القصيرة في تليجرام (`/w/...`) واللوجو في الإيميل يشتغلوا.

## رسايل العميل (3 مراحل) وتنبيهات الأدمن
- رسالة الأدمن/تليجرام للطلب الجديد **زي ما هي** (مفيش تغيير).
- **1) تأكيد الطلب:** رسالة واتساب + إيميل طويلة ببريميوم وفيها تفاصيل (القطع، الملخص، العنوان، الخطوات الجاية، ملاحظات). الإيميل بيستخدم نفس خطوط الموقع (Noto Kufi Arabic للجملة، Sora للإنجليزي). الواتساب ما بيدعمش خطوط، فبيظهر بخط واتساب نفسه.
- **2) اتسلّم لشركة الشحن:** لما تغيّر الحالة في Admin → orders لـ `shipped` بتوصلك رسالة تليجرام (لكل الحسابات) فيها لينك بيفتح واتساب العميل والرسالة مكتوبة. والعميل بيجيله إيميل لو سايب إيميل.
- **3) خرج للتوصيل:** حالة جديدة `out_for_delivery`. نفس الفكرة: تليجرام + لينك واتساب + إيميل.
- **يوم التوصيل:** الساعة 9 الصبح (توقيت القاهرة) في اليوم الثالث من وقت الأوردر (يعني بعد يومين) بتجيلك رسالة تليجرام لكل أوردر لسه `confirmed` أو `shipped` فيها بيانات العميل والعنوان ولينك رسالة «خرج للتوصيل». بتتبعت مرة واحدة لكل أوردر.
- في Admin → orders كل أوردر عليه 3 أزرار واتساب: `1 · Confirmed` و`2 · Handed to courier` و`3 · Out for delivery`.
- ما بتتبعتش تلقائيًا من غير ضغطتك على واتساب، لأن الإرسال التلقائي للعملاء عن طريق WhatsApp Cloud API مدفوع.

## جدول المقاسات من الأدمن
Admin → settings → *Size chart*: عدّل أي خانة، زوّد/امسح مقاسات (صفوف) وقياسات (أعمدة)، واكتب الملاحظة اللي تحت الجدول، ثم *Save size chart*. بيظهر فورًا في صفحة Size guide.
#   F I K R _ S T O R E  
 