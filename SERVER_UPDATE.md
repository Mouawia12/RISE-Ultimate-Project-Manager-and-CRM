# Server Update Commands (Safe)

Use these commands on the VPS to update RISE safely without breaking runtime config.

## 1) Safe update (recommended)

```bash
git -C /opt/src/rise fetch --all
git -C /opt/src/rise reset --hard origin/main

rsync -a \
  --exclude='.env' \
  --exclude='writable/' \
  --exclude='app/Config/Database.php' \
  /opt/src/rise/ /var/www/portal/

systemctl reload nginx
systemctl restart php8.3-fpm || systemctl restart php-fpm
```

## 2) Quick health check

```bash
curl -I https://portal.bawaderamal.com.sa
```

## 3) Do NOT use these in production

Avoid using:

```bash
rsync -a --delete /opt/src/rise/ /var/www/portal/
```

Reason: it may remove or overwrite critical runtime files (`.env`, `writable`, DB config) and break the portal.

## 4) If you changed code locally

Push first from local machine:

```bash
cd /Users/mw/Downloads/RISE-Ultimate-Project-Manager-and-CRM
git add -A
git commit -m "your message"
git push origin main
```

Then run section (1) on the server.

