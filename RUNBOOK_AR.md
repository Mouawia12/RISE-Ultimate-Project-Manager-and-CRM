# دليل التشغيل والنشر (بالعربية)

هذا الملف يلخص طريقة إدارة ونشر النظام الحالي على السيرفر.

## 1) نظرة عامة
1. الموقع الرئيسي (Next.js) يعمل عبر PM2 باسم `user-frontend` على المنفذ `3002`.
2. الـ API الرئيسي (NestJS) يعمل عبر PM2 باسم `user-backend` على المنفذ `3011`.
3. بوابة الموظفين RISE تعمل على:
`https://portal.bawaderamal.com.sa`
4. Nginx هو بوابة الويب (80/443) مع PHP-FPM.
5. قاعدة بيانات RISE هي: `rise_portal`.

## 2) المسارات المهمة (VPS)
1. مسار البوابة الحي: `/var/www/portal`
2. مسار السورس المسحوب من GitHub: `/opt/src/rise`
3. مسار مشروع الموقع الرئيسي: `/root/projects/User/BawaderFrontSite`
4. إعداد Nginx للبوابة: `/etc/nginx/sites-available/portal-bawaderamal.conf`
5. الشهادات: `/etc/letsencrypt/live/portal.bawaderamal.com.sa/`
6. لوجات RISE: `/var/www/portal/writable/logs/`
7. لوج Nginx: `/var/log/nginx/error.log`

## 3) السحب الأول من GitHub (RISE)
```bash
apt update && apt install -y git rsync
mkdir -p /opt/src
git clone https://github.com/Mouawia12/RISE-Ultimate-Project-Manager-and-CRM.git /opt/src/rise

mkdir -p /var/www/portal
rsync -a --delete /opt/src/rise/ /var/www/portal/
chown -R www-data:www-data /var/www/portal
find /var/www/portal -type d -exec chmod 755 {} \;
find /var/www/portal -type f -exec chmod 644 {} \;
chmod -R 775 /var/www/portal/writable /var/www/portal/files

nginx -t && systemctl reload nginx
systemctl restart php8.3-fpm || systemctl restart php-fpm
```

## 4) تحديث RISE لاحقًا
```bash
git -C /opt/src/rise fetch --all
git -C /opt/src/rise reset --hard origin/main

rsync -a --delete /opt/src/rise/ /var/www/portal/
chown -R www-data:www-data /var/www/portal
chmod -R 775 /var/www/portal/writable /var/www/portal/files

nginx -t && systemctl reload nginx
systemctl restart php8.3-fpm || systemctl restart php-fpm
```

## 5) قاعدة البيانات
إنشاء قاعدة ومستخدم:
```bash
mysql -uroot -e "
CREATE DATABASE IF NOT EXISTS rise_portal CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
CREATE USER IF NOT EXISTS 'rise_user'@'localhost' IDENTIFIED BY 'CHANGE_ME_PASSWORD';
GRANT ALL PRIVILEGES ON rise_portal.* TO 'rise_user'@'localhost';
FLUSH PRIVILEGES;"
```

استيراد البيانات:
```bash
mysql -uroot rise_portal < /var/www/portal/install.disabled/database.sql
```

## 6) ملف إعداد البيئة
المسار:
`/var/www/portal/.env`

وضع الإنتاج:
```ini
CI_ENVIRONMENT = production
app.baseURL = 'https://portal.bawaderamal.com.sa/'
```

ثم:
```bash
systemctl restart php8.3-fpm || systemctl restart php-fpm
systemctl reload nginx
```

## 7) الترخيص
إذا ظهر خطأ ترخيص:
1. تأكد من صحة `item_purchase_code`.
2. تأكد أن السيرفر يصل إلى `https://releases.fairsketch.com/rise/`.
3. إذا الكود مستخدم على دومين آخر، عطّل الترخيص هناك أولًا (Disable License) ثم فعّله هنا.

فحص سريع:
```bash
mysql -uroot rise_portal -e "SELECT setting_name,setting_value FROM settings WHERE setting_name IN ('item_purchase_code','app_verification_key','disable_installation');"
```

## 8) استرجاع دخول الأدمن
إذا تعذر تسجيل الدخول، يمكن ضبط أول مستخدم كأدمن عبر SQL (بحذر).

## 9) تعديل رابط البوابة في الفوتر (الموقع الرئيسي)
المجلد:
`/root/projects/User/BawaderFrontSite`

الملفات:
1. `src/components/layout/Footer/index.tsx`
2. `src/messages/footer/ar.json`
3. `src/messages/footer/en.json`

بعد التعديل:
```bash
cd /root/projects/User/BawaderFrontSite
npm install
npm run build
pm2 restart user-frontend
pm2 save
```

## 10) فحوصات بعد أي تحديث
```bash
systemctl status nginx --no-pager
systemctl status php8.3-fpm --no-pager || systemctl status php-fpm --no-pager
pm2 list

curl -I https://portal.bawaderamal.com.sa
curl -I https://bawaderamal.com.sa

tail -n 100 /var/log/nginx/error.log
tail -n 100 /var/www/portal/writable/logs/*.log
```

## 11) نسخ احتياطي
نسخة قاعدة البيانات:
```bash
mysqldump -uroot rise_portal > /root/backup-rise_portal-$(date +%F-%H%M).sql
```

نسخة ملفات البوابة:
```bash
tar -czf /root/backup-portal-files-$(date +%F-%H%M).tar.gz /var/www/portal
```

## 12) ملاحظات مهمة
1. لا تنشر البوابة على `public_html` في هذه البنية.
2. التشغيل الفعلي يتم على VPS.
3. لا تشارك كود الشراء في أي مكان عام.
4. بعد أي تصحيح، أعد البيئة إلى `production`.
