# Развёртывание Integram на VPS

Два скрипта: `setup-server.sh` настраивает чистый Debian 12/13, `deploy-prod.sh` выкладывает и обновляет код.

---

## Быстрый старт

```bash
# 1. На ЧИСТОМ Debian 12/13 VPS (от root):
DOMAIN=your-domain.com bash scripts/setup-server.sh

# 2. С локальной машины — первая выкладка:
DEPLOY_HOST=root@YOUR_IP bash scripts/deploy-prod.sh setup

# 3. Дальнейшие обновления:
bash scripts/deploy-prod.sh all
```

---

## Требования к серверу

- Debian 12 (Bookworm) или Debian 13 (Trixie), Ubuntu 22.04+
- 4+ CPU, 4+ GB RAM (минимум 2 CPU / 2 GB — тесно)
- 50+ GB диск (NVMe предпочтительно)
- Домен с DNS A-записью на IP сервера

## Скрипт 1: setup-server.sh

Полная настройка чистого VPS от root. Идемпотентен — повторный запуск безопасен.

### Что устанавливает

| Компонент | Версия | Зачем |
|---|---|---|
| Node.js | 24 LTS | Backend, Frontend build |
| Bun | latest | BullMQ worker |
| PM2 | latest | Process manager + autostart |
| PostgreSQL | 17 (pgdg) | БД + pgvector + pg_trgm |
| Redis | 7 | Очереди, кэш, сессии |
| Nginx | latest | Reverse proxy, SSL termination, static |
| Docker | latest | Machine Gate CI sandbox |
| Certbot | latest | Let's Encrypt SSL |

### Hardening (безопасность)

- **SSH**: только ключи, root по ключу, MaxAuthTries 4
- **UFW**: deny incoming, allow SSH/80/443
- **Fail2Ban**: SSH jail, ban 3h после 3 попыток
- **unattended-upgrades**: автоматические security-патчи
- **sysctl**: tcp_tw_reuse, somaxconn 65535, rp_filter
- **Docker isolation**: контейнеры не видят хост (iptables DOCKER-USER)
- **nofile**: 65535 для всех пользователей

### Что создаёт

```
/opt/integram/                 — код приложения
/opt/integram/shared/kit/      — @kit артефакты (ВНЕ rsync --delete)
/opt/integram/ecosystem.config.cjs — PM2 конфигурация
/var/integram/uploads/         — загруженные файлы
/var/integram/data/            — данные
/var/integram/temp/            — временные файлы
/var/integram/backups/         — бэкапы БД и файлов
/var/lib/integram/workspaces/  — AI workspaces
/var/log/integram/             — логи (ротация 14 дней)
```

### Переменные окружения

```bash
DOMAIN=example.com       # домен для SSL (пусто = без SSL)
DEPLOY_USER=deploy       # пользователь (по умолчанию deploy)
SSH_PORT=22              # SSH порт
PG_VERSION=17            # версия PostgreSQL
PG_DB=integram           # имя БД
PG_USER=integram         # пользователь БД
PG_PASS=...              # пароль БД (генерируется автоматически)
REDIS_PASS=...           # пароль Redis (генерируется автоматически)
```

### Бэкапы (автоматические)

- **БД**: ежедневно в 3:00, `pg_dump -Fc`, хранение 7 дней
- **Файлы**: по воскресеньям в 4:00, tar.gz, хранение 30 дней
- Ротация через cron (`/etc/cron.d/integram-backup`)

### После установки

Скрипт выводит сгенерированные пароли и сохраняет их в `/root/.integram-credentials` (chmod 600). Перенесите в `.env` и удалите файл.

---

## Скрипт 2: deploy-prod.sh

Выкладка с локальной машины на сервер через rsync + SSH.

### Команды

| Команда | Что делает |
|---|---|
| `setup` | Первая установка: sync всего, npm install, build, PM2 start |
| `all` | Фронт + бэк + portal + restart + health check |
| `front` | Только frontend (backend/public, с `--delete` и сторожем) |
| `back` | Только backend/src + scripts + restart integram/worker |
| `portal` | Build portal + sync .output + restart portal |
| `deps` | npm/bun install на сервере + restart all |
| `health` | Состояние: PM2, сервисы, API health, диск, RAM |

### Переменные

```bash
DEPLOY_HOST=root@178.253.39.215   # SSH-адрес сервера
SKIP_BUILD=1                       # пропустить локальную сборку
```

### Примеры

```bash
# Обычное обновление (фронт + бэк)
cd frontend && npx vite build && cd ..
bash scripts/deploy-prod.sh all

# Только бэкенд (без сборки)
bash scripts/deploy-prod.sh back

# Только portal
bash scripts/deploy-prod.sh portal

# Обновить зависимости
bash scripts/deploy-prod.sh deps

# Проверить состояние
bash scripts/deploy-prod.sh health

# Выкладка на другой сервер
DEPLOY_HOST=root@NEW_IP bash scripts/deploy-prod.sh setup
```

### Сторож удалений

`rsync --delete` сносит на сервере всё, чего нет в источнике. Так дважды (12–13.08.2026) пропадали версии @kit, к которым привязаны порталы.

Защита — два слоя:

1. **Артефакты вне rsync**: @kit живёт в `/opt/integram/shared/kit/`, отдаётся отдельным правилом nginx (`location /assets/kit/` ВЫШЕ общего). rsync до них не дотягивается.

2. **Сторож**: выкладка фронта сперва идёт вхолостую (`--dry-run -v`). Если есть удаления — показывает перечень и требует подтверждения словом «удалить».

---

## .env — обязательные поля

```bash
cp backend/.env.example backend/.env
nano backend/.env
```

**Минимум для запуска:**

```env
NODE_ENV=production
PORT=8081
HOST=127.0.0.1

DATABASE_URL=postgresql://integram:ПАРОЛЬ@localhost:5432/integram
DB_HOST=localhost
DB_PORT=5432
DB_NAME=integram
DB_USER=integram
DB_PASSWORD=ПАРОЛЬ

REDIS_HOST=127.0.0.1
REDIS_PORT=6379
REDIS_PASSWORD=ПАРОЛЬ_REDIS

# Сгенерировать!
JWT_SECRET=           # node -e "console.log(require('crypto').randomBytes(64).toString('hex'))"
SESSION_SECRET=       # node -e "console.log(require('crypto').randomBytes(32).toString('hex'))"
INTEGRAM_PHP_SALT=    # openssl rand -base64 24

UPLOAD_DIR=/var/integram/uploads
DATA_DIR=/var/integram/data
TEMP_DIR=/var/integram/temp
WORKSPACE_ROOT=/var/lib/integram/workspaces

FRONTEND_URL=https://your-domain.com
CORS_ORIGIN=https://your-domain.com
```

Полный перечень — в `backend/.env.example`.

---

## Nginx

Конфиг генерируется `setup-server.sh` в `/etc/nginx/sites-available/integram`.

Ключевые блоки:
- `/api/` → backend:8081 (WebSocket upgrade, SSE, 1h timeout)
- `/assets/kit/` → `/opt/integram/shared/kit/` (immutable, 1y cache)
- `~^/[^/]+/portal` → portal:3000 (Nuxt SSR)
- `/` → `backend/public` (Vue SPA, try_files, хешированные ассеты 1y)
- gzip для text/css/js/json/svg

SSL добавляется certbot:
```bash
certbot --nginx -d your-domain.com
```

---

## PM2

Процессы (ecosystem.config.cjs):

| Процесс | Скрипт | Порт | Описание |
|---|---|---|---|
| integram | `backend/scripts/start-pg.js` | 8081 | API, WebSocket, SSE |
| worker | `backend/scripts/start-worker.js` | — | BullMQ (bun) |
| portal | `portal/.output/server/index.mjs` | 3000 | Nuxt SSR |
| browser | `browser/index.js` | 3099 | Playwright scraper |

```bash
pm2 status                    # все процессы
pm2 logs integram --lines 50  # логи
pm2 restart integram          # перезапуск
pm2 monit                     # мониторинг
```

Автостарт при ребуте: `pm2 startup` + `pm2 save`.

---

## Проверка

```bash
# На сервере
pm2 status
systemctl status postgresql redis-server nginx
curl http://localhost:8081/api/v2/health
ss -tlnp | grep -E '8081|3000|3099|5432|6379'

# С локальной машины
bash scripts/deploy-prod.sh health
```

---

## Первый пользователь

Через UI (`https://your-domain.com`) или API:

```bash
curl -X POST https://your-domain.com/api/v2/iam/register \
  -H 'Content-Type: application/json' \
  -d '{"username":"admin","email":"admin@your-domain.com","password":"strong-password"}'
```

---

## Устранение проблем

```bash
# Логи
pm2 logs integram --lines 100
pm2 logs worker --lines 50
journalctl -u postgresql -n 50
tail -f /var/log/nginx/error.log

# Redis
redis-cli -a ПАРОЛЬ ping    # PONG

# PostgreSQL
sudo -u postgres psql -d integram -c "SELECT 1;"

# Nginx
nginx -t
systemctl reload nginx

# Перезапуск всего
pm2 restart all
```
