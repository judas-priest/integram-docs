# Развёртывание Integram на VPS

Пошаговая инструкция: поднять Integram на чистом сервере с нуля.

Все команды выполняются от root, если не указано иное.

---

## Требования к серверу

- Debian 12+ / Ubuntu 22.04+
- 4+ CPU, 4+ GB RAM (минимум 2 CPU / 2 GB — но будет тесно)
- 50+ GB диск
- Домен с DNS (A-запись на IP сервера)

## 1. Установить зависимости

```bash
apt-get update && apt-get upgrade -y

# Node.js 24
curl -fsSL https://deb.nodesource.com/setup_24.x | bash -
apt-get install -y nodejs

# Bun (для BullMQ worker)
curl -fsSL https://bun.sh/install | bash
source ~/.bashrc

# PostgreSQL 16 + pgvector
apt-get install -y postgresql-16 postgresql-16-pgvector

# Redis 7
apt-get install -y redis-server

# PM2
npm install -g pm2

# Nginx + SSL
apt-get install -y nginx certbot python3-certbot-nginx

# Docker (для Machine Gate CI sandbox)
apt-get install -y docker.io
systemctl enable docker
docker pull node:22-slim

# Блокировать контейнерам доступ к хосту
iptables -I DOCKER-USER -s 172.17.0.0/16 -d 172.17.0.1 -j DROP
apt-get install -y iptables-persistent
```

## 2. Настроить PostgreSQL

```bash
sudo -u postgres psql <<'SQL'
CREATE DATABASE integram;
CREATE USER integram WITH PASSWORD 'ВАША_СИЛЬНАЯ_ПАРОЛЬ';
GRANT ALL PRIVILEGES ON DATABASE integram TO integram;
ALTER DATABASE integram OWNER TO integram;
\c integram
CREATE EXTENSION IF NOT EXISTS vector;
CREATE EXTENSION IF NOT EXISTS pg_trgm;
SQL
```

Проверить:
```bash
sudo -u postgres psql -d integram -c "SELECT extname FROM pg_extension;"
# Должны быть: plpgsql, vector, pg_trgm
```

## 3. Настроить Redis

```bash
# Установить пароль
sed -i 's/# requirepass foobared/requirepass ВАША_ПАРОЛЬ_REDIS/' /etc/redis/redis.conf
# Включить AOF
sed -i 's/appendonly no/appendonly yes/' /etc/redis/redis.conf
systemctl restart redis
```

## 4. Создать директории

```bash
mkdir -p /opt/integram
mkdir -p /var/integram/{uploads,data,temp,backups}
mkdir -p /var/lib/integram/workspaces
mkdir -p /var/log/integram
```

## 5. Получить код

**Вариант A — из Git:**
```bash
cd /opt
git clone https://github.com/judas-priest/integram.git
cd integram
```

**Вариант B — через rsync (если нет доступа к репо):**
```bash
# На локальной машине:
rsync -az --exclude node_modules --exclude .git \
  ./ root@СЕРВЕР:/opt/integram/
```

## 6. Установить зависимости и собрать

```bash
cd /opt/integram

# Backend
cd backend
PUPPETEER_SKIP_DOWNLOAD=true PLAYWRIGHT_SKIP_BROWSER_DOWNLOAD=1 \
  npm install --omit=dev --legacy-peer-deps
cd ..

# Frontend (собирается в backend/public)
cd frontend
npm ci
npx vite build
cd ..

# Portal
cd portal
npm ci
npx nuxt build
cd ..

# Browser (Playwright scraper)
cd browser
npm install --omit=dev
npx playwright install chromium --with-deps
cd ..

# Worker (BullMQ) — зависимости берёт из backend
cd backend
bun install --production
cd ..
```

## 7. Настроить .env

```bash
cp backend/.env.example backend/.env
nano backend/.env
```

**Обязательные поля — заполнить:**

```env
# ─── Сервер ───────────────────────────────────────────────
NODE_ENV=production
PORT=8081
HOST=127.0.0.1
HTTPS_ENABLED=false

# ─── База данных ──────────────────────────────────────────
DATABASE_URL=postgresql://integram:ВАША_ПАРОЛЬ@localhost:5432/integram
DB_HOST=localhost
DB_PORT=5432
DB_NAME=integram
DB_USER=integram
DB_PASSWORD=ВАША_ПАРОЛЬ

# ─── Redis ────────────────────────────────────────────────
REDIS_HOST=127.0.0.1
REDIS_PORT=6379
REDIS_PASSWORD=ВАША_ПАРОЛЬ_REDIS

# ─── Безопасность (сгенерировать!) ────────────────────────
# НИКОГДА не использовать значения по умолчанию в продакшене
JWT_SECRET=            # node -e "console.log(require('crypto').randomBytes(64).toString('hex'))"
SESSION_SECRET=        # node -e "console.log(require('crypto').randomBytes(32).toString('hex'))"
INTEGRAM_PHP_SALT=     # openssl rand -base64 24

# ─── Директории ──────────────────────────────────────────
UPLOAD_DIR=/var/integram/uploads
DATA_DIR=/var/integram/data
TEMP_DIR=/var/integram/temp
WORKSPACE_ROOT=/var/lib/integram/workspaces
LOCAL_BACKUP_PATH=/var/integram/backups
LOG_LEVEL=info

# ─── AI (минимум один провайдер) ──────────────────────────
# Без AI-ключа платформа работает, но чат и агенты недоступны
POLZA_AI_API_KEY=
DEEPSEEK_API_KEY=
ANTHROPIC_API_KEY=
OPENAI_API_KEY=

# ─── Домен ────────────────────────────────────────────────
FRONTEND_URL=https://ваш-домен.com
CORS_ORIGIN=https://ваш-домен.com

# ─── Telegram бот (опционально) ──────────────────────────
TELEGRAM_BOT_TOKEN=

# ─── Email (опционально) ─────────────────────────────────
SMTP_HOST=smtp.gmail.com
SMTP_PORT=587
SMTP_SECURE=false
SMTP_USER=
SMTP_PASSWORD=
FROM_EMAIL=noreply@ваш-домен.com
FROM_NAME=Integram

# ─── Регистрация пользователей ───────────────────────────
INTEGRAM_REGISTRATION_USERNAME=admin
INTEGRAM_REGISTRATION_PASSWORD=    # сгенерировать сильный пароль

# ─── Бэкапы ──────────────────────────────────────────────
BACKUP_ENABLED=true
BACKUP_ENCRYPTION_ENABLED=true
BACKUP_ENCRYPTION_KEY=             # openssl rand -base64 32

# ─── Платёжные системы (по необходимости) ─────────────────
# STRIPE_API_KEY=
# YOOKASSA_SHOP_ID=
# YOOKASSA_SECRET_KEY=
# ROBOKASSA_MERCHANT_ID=
# TINKOFF_TERMINAL_KEY=
```

## 8. Запустить через PM2

```bash
cd /opt/integram

cat > ecosystem.config.cjs <<'PMEOF'
'use strict';
module.exports = {
  apps: [
    {
      name: 'integram',
      script: 'backend/scripts/start-pg.js',
      cwd: '/opt/integram',
      node_args: '--max-old-space-size=2048',
      env: { NODE_ENV: 'production' },
      out_file: '/var/log/integram/backend-out.log',
      error_file: '/var/log/integram/backend-err.log',
    },
    {
      name: 'worker',
      script: 'backend/scripts/start-worker.js',
      cwd: '/opt/integram',
      interpreter: 'bun',
      env: { NODE_ENV: 'production' },
      out_file: '/var/log/integram/worker-out.log',
      error_file: '/var/log/integram/worker-err.log',
    },
    {
      name: 'portal',
      script: 'portal/.output/server/index.mjs',
      cwd: '/opt/integram',
      env: {
        NODE_ENV: 'production',
        PORT: 3000,
        API_BASE_URL: 'http://127.0.0.1:8081',
      },
      out_file: '/var/log/integram/portal-out.log',
      error_file: '/var/log/integram/portal-err.log',
    },
    {
      name: 'signal',
      script: 'backend/signal-server',
      cwd: '/opt/integram',
      env: { NODE_ENV: 'production' },
      out_file: '/var/log/integram/signal-out.log',
      error_file: '/var/log/integram/signal-err.log',
    },
    {
      name: 'browser',
      script: 'browser/index.js',
      cwd: '/opt/integram',
      env: { NODE_ENV: 'production' },
      out_file: '/var/log/integram/browser-out.log',
      error_file: '/var/log/integram/browser-err.log',
    },
  ],
};
PMEOF

pm2 start ecosystem.config.cjs
pm2 save
pm2 startup
```

## 9. Nginx

Заменить `ваш-домен.com` на ваш домен:

```bash
cat > /etc/nginx/sites-available/integram <<'NGINX'
server {
    listen 80;
    server_name ваш-домен.com;
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl http2;
    server_name ваш-домен.com;

    ssl_certificate     /etc/letsencrypt/live/ваш-домен.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/ваш-домен.com/privkey.pem;

    client_max_body_size 100M;

    # API + WebSocket + SSE
    location /api/ {
        proxy_pass http://127.0.0.1:8081;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_read_timeout 3600s;
        proxy_send_timeout 3600s;
        proxy_buffering off;        # для SSE (AI чат)
    }

    # Portal (Nuxt SSR)
    location ~ ^/[^/]+/portal {
        proxy_pass http://127.0.0.1:3000;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    # Vue SPA (статика)
    location / {
        root /opt/integram/backend/public;
        try_files $uri $uri/ /index.html;
        expires 1h;
    }
}
NGINX

ln -sf /etc/nginx/sites-available/integram /etc/nginx/sites-enabled/
rm -f /etc/nginx/sites-enabled/default

# SSL-сертификат (сначала без SSL, потом certbot добавит)
# Временно убрать ssl-строки, получить сертификат, вернуть:
certbot --nginx -d ваш-домен.com

nginx -t && systemctl reload nginx
```

## 10. Проверка

```bash
# Все процессы работают?
pm2 status
# integram     │ online
# worker       │ online
# portal       │ online
# browser      │ online

# Сервисы?
systemctl status postgresql redis nginx

# Health check
curl http://localhost:8081/api/v2/health
# {"ok":true,"data":{"status":"healthy",...}}

# Через браузер
# https://ваш-домен.com — должна открыться страница логина
```

## 11. Создать первого пользователя

Зарегистрировать через UI (`https://ваш-домен.com`) или через API:

```bash
curl -X POST https://ваш-домен.com/api/v2/iam/register \
  -H 'Content-Type: application/json' \
  -d '{
    "username": "admin",
    "email": "admin@ваш-домен.com",
    "password": "сильный-пароль-123"
  }'
```

---

## Обновление

```bash
cd /opt/integram
git pull origin master

# Пересобрать frontend
cd frontend && npx vite build && cd ..

# Пересобрать portal
cd portal && npx nuxt build && cd ..

# Обновить зависимости backend (если изменились)
cd backend && npm install --omit=dev --legacy-peer-deps && cd ..

# Перезапустить
pm2 restart all
```

Или через rsync с локальной машины:
```bash
# Frontend (собрать локально, залить)
cd frontend && npx vite build && cd ..
rsync -az backend/public/ root@СЕРВЕР:/opt/integram/backend/public/

# Backend код
rsync -az backend/src/ root@СЕРВЕР:/opt/integram/backend/src/
rsync -az backend/scripts/ root@СЕРВЕР:/opt/integram/backend/scripts/

# Portal
cd portal && npx nuxt build && cd ..
rsync -az --delete portal/.output/ root@СЕРВЕР:/opt/integram/portal/.output/

ssh СЕРВЕР 'pm2 restart all'
```

## Бэкапы

Бэкап базы данных:
```bash
pg_dump -Fc -U integram -d integram -f /var/integram/backups/integram-$(date +%Y%m%d).dump
```

Бэкап файлов:
```bash
tar czf /var/integram/backups/files-$(date +%Y%m%d).tar.gz \
  /var/integram/uploads /var/integram/data /var/lib/integram/workspaces
```

Автоматически (cron):
```bash
crontab -e
# Каждый день в 3:00
0 3 * * * pg_dump -Fc -U integram -d integram -f /var/integram/backups/integram-$(date +\%Y\%m\%d).dump
0 3 * * * find /var/integram/backups -mtime +7 -delete
```

## Устранение проблем

```bash
# Логи backend
pm2 logs integram --lines 50

# Логи worker
pm2 logs worker --lines 50

# Логи portal
pm2 logs portal --lines 50

# PostgreSQL
journalctl -u postgresql -n 50

# Redis
redis-cli -a ПАРОЛЬ ping
# PONG

# Nginx
nginx -t
tail -f /var/log/nginx/error.log

# Перезапуск всего
pm2 restart all

# Проверить порты
ss -tlnp | grep -E '8081|3000|3099|5432|6379'
```
