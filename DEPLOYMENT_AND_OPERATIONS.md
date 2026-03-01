# Deployment and Operations Guide

This document explains how the current system is deployed and maintained.

## 1) Current Architecture
1. Main website frontend: Next.js app on VPS, managed by PM2 (`user-frontend` on port `3002`).
2. Main website backend: NestJS app on VPS, managed by PM2 (`user-backend` on port `3011`).
3. Employee portal: RISE (PHP/CodeIgniter) on `https://portal.bawaderamal.com.sa`.
4. Web server: Nginx handles HTTPS, reverse proxy, and PHP-FPM.
5. Portal database: MariaDB/MySQL database `rise_portal`.

## 2) Important Paths (VPS)
1. Live RISE portal path: `/var/www/portal`
2. RISE source clone path: `/opt/src/rise`
3. Main website frontend path: `/root/projects/User/BawaderFrontSite`
4. Nginx portal vhost: `/etc/nginx/sites-available/portal-bawaderamal.conf`
5. Enabled Nginx link: `/etc/nginx/sites-enabled/portal-bawaderamal.conf`
6. SSL certificates: `/etc/letsencrypt/live/portal.bawaderamal.com.sa/`
7. RISE logs: `/var/www/portal/writable/logs/`
8. Nginx error log: `/var/log/nginx/error.log`

## 3) First-Time Pull From GitHub (RISE)
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

## 4) Update RISE Later (Git Pull + Deploy)
```bash
git -C /opt/src/rise fetch --all
git -C /opt/src/rise reset --hard origin/main

rsync -a --delete /opt/src/rise/ /var/www/portal/
chown -R www-data:www-data /var/www/portal
chmod -R 775 /var/www/portal/writable /var/www/portal/files

nginx -t && systemctl reload nginx
systemctl restart php8.3-fpm || systemctl restart php-fpm
```

## 5) Database Setup For RISE
```bash
mysql -uroot -e "
CREATE DATABASE IF NOT EXISTS rise_portal CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
CREATE USER IF NOT EXISTS 'rise_user'@'localhost' IDENTIFIED BY 'CHANGE_ME_PASSWORD';
GRANT ALL PRIVILEGES ON rise_portal.* TO 'rise_user'@'localhost';
FLUSH PRIVILEGES;"

mysql -uroot rise_portal < /var/www/portal/install.disabled/database.sql
```

Then configure `/var/www/portal/app/Config/Database.php`:
1. `hostname`
2. `username`
3. `password`
4. `database`
5. `DBPrefix` must match actual table names (`rise_` or empty).

## 6) Environment Mode
RISE `.env` file path:
`/var/www/portal/.env`

Production mode:
```ini
CI_ENVIRONMENT = production
app.baseURL = 'https://portal.bawaderamal.com.sa/'
```

After any `.env` change:
```bash
systemctl restart php8.3-fpm || systemctl restart php-fpm
systemctl reload nginx
```

## 7) License Notes
License is checked against FairSketch server.
If you see `verification_failed` or `unable to verify your license`:
1. Ensure `item_purchase_code` is correct.
2. Ensure server can access `https://releases.fairsketch.com/rise/`.
3. If code was used on another domain, disable license there first, then activate here.

Key settings are in table `settings`:
1. `item_purchase_code`
2. `app_verification_key`
3. `disable_installation`

Quick check:
```bash
mysql -uroot rise_portal -e "SELECT setting_name,setting_value FROM settings WHERE setting_name IN ('item_purchase_code','app_verification_key','disable_installation');"
```

## 8) Admin Access Recovery (RISE)
If login fails, set first user to active admin.

```bash
DB_NAME="rise_portal"
ADMIN_EMAIL="admin@bawaderamal.com.sa"
ADMIN_PASS='Admin@123456'
HASH=$(php -r "echo password_hash('$ADMIN_PASS', PASSWORD_DEFAULT);")

if mysql -uroot "$DB_NAME" -Nse "SHOW TABLES LIKE 'rise_users';" | grep -q rise_users; then
  USERS_TABLE="rise_users"
else
  USERS_TABLE="users"
fi

mysql -uroot "$DB_NAME" -e "
UPDATE ${USERS_TABLE}
SET
  email='${ADMIN_EMAIL}',
  password='${HASH}',
  user_type='staff',
  is_admin=1,
  status='active',
  deleted=0,
  disable_login=0
WHERE id = (
  SELECT id FROM (
    SELECT id FROM ${USERS_TABLE} ORDER BY id ASC LIMIT 1
  ) t
);"
```

## 9) Main Website Footer Link Work (Bilingual)
Frontend repo path:
`/root/projects/User/BawaderFrontSite`

Footer file:
`src/components/layout/Footer/index.tsx`

Translations:
1. `src/messages/footer/ar.json`
2. `src/messages/footer/en.json`

Add translation key `quickLinks.employeePortal` in both files.
Add footer link to `https://portal.bawaderamal.com.sa` below About link.

Build and restart:
```bash
cd /root/projects/User/BawaderFrontSite
npm install
npm run build
pm2 restart user-frontend
pm2 save
```

## 10) Nginx Redirect Shortcut (Optional)
Optional shortcut on main site:
`https://bawaderamal.com.sa/employee-portal` -> `https://portal.bawaderamal.com.sa`

Add location rule in main site nginx config, then:
```bash
nginx -t && systemctl reload nginx
```

## 11) Health Checks After Any Change
```bash
systemctl status nginx --no-pager
systemctl status php8.3-fpm --no-pager || systemctl status php-fpm --no-pager
pm2 list

curl -I https://portal.bawaderamal.com.sa
curl -I https://bawaderamal.com.sa

tail -n 100 /var/log/nginx/error.log
tail -n 100 /var/www/portal/writable/logs/*.log
```

## 12) Backup Commands
Database backup:
```bash
mysqldump -uroot rise_portal > /root/backup-rise_portal-$(date +%F-%H%M).sql
```

Portal files backup:
```bash
tar -czf /root/backup-portal-files-$(date +%F-%H%M).tar.gz /var/www/portal
```

## 13) Important Operational Notes
1. Do not deploy RISE in shared hosting `public_html` for this setup.
2. Production is running on VPS.
3. Avoid posting purchase code in chat or public repos.
4. After fixing any issue, switch environment back to production.
