# Showcase

Витрина студенческих проектов. Монорепо с двумя сабмодулями:

- `showcase-backend/` — Spring Boot 3 / Java 23 REST API.
- `showcase-frontend/` — React 19 + Vite SPA.

Подробности по архитектуре кода — в [CLAUDE.md](CLAUDE.md), по деплою — в [DEPLOY.md](DEPLOY.md).

## Клонирование и обновление с сабмодулями

```bash
# первый клон
git clone --recurse-submodules <repo-url>
cd showcase

# если уже клонировали без сабмодулей
git submodule update --init --recursive

# обновить и корень, и сабмодули
git pull --recurse-submodules
git submodule update --remote --merge   # подтянуть свежие коммиты сабмодулей
```

Полезно один раз настроить:

```bash
git config --global submodule.recurse true
```

Тогда `git pull` / `git checkout` будут автоматически обновлять сабмодули.

## Локальный запуск через docker-compose

Нужно: Docker Desktop (или Docker Engine + Compose v2).

```bash
cp .env.example .env
# отредактируй .env: пароли, APP_DEPLOYMENT_MODE и т.д.

docker compose up --build
```

Стек слушает `http://127.0.0.1:8081` (порт задаётся `HTTP_PORT` в `.env`).
Открой `http://localhost:8081`.

### Минимум переменных в `.env` для локального запуска

- `POSTGRES_PASSWORD` — любой непустой.
- `MINIO_ROOT_USER` / `MINIO_ROOT_PASSWORD` — любые.
- `FRONT_URL=http://localhost:8081` (для локалки, без HTTPS).
- `APP_DEPLOYMENT_MODE=dev` — чтобы обойти Azure OAuth.
- `DEV_AUTH_TOKEN=<любой секрет>` — токен для Bearer-аутентификации.
- `DEV_USER_ID=<id из таблицы users>` — от чьего имени работают запросы.

В dev-режиме на странице `/login` появляется поле для ввода токена; глобальный
fetch-интерсептор добавляет `Authorization: Bearer <token>` ко всем запросам.

### Что входит в стек

| Сервис        | Назначение                                                   |
|---------------|--------------------------------------------------------------|
| `postgres`    | Хранилище данных (volume `pgdata`).                          |
| `minio`       | S3-совместимое хранилище файлов (volume `minio`).            |
| `minio-init`  | One-shot: создаёт бакет `S3_BUCKET_NAME`.                    |
| `backend`     | Spring Boot API, слушает `:8080` внутри сети.                |
| `frontend`    | One-shot build SPA → копирует `dist/` в volume.              |
| `nginx`       | Раздаёт SPA + проксирует API на `backend:8080`.              |

Наружу публикуется только `nginx` на `127.0.0.1:${HTTP_PORT}`.

### Полезные команды

```bash
docker compose logs -f backend
docker compose build frontend && docker compose up -d nginx   # пересобрать фронт
docker compose down            # стоп
docker compose down -v         # стоп + удалить данные (БД, MinIO)
```

---

# Архитектура развёртывания на сервере с публичным IP / доменом

```
                        ┌─────────────────────────────────────────────┐
   Internet ── :443 ──► │ host nginx (на VM)                          │
   (HTTPS)              │  - TLS-терминация (Let's Encrypt / certbot) │
                        │  - HTTP→HTTPS redirect                      │
                        │  - proxy_pass → 127.0.0.1:8081              │
                        │  - проставляет X-Forwarded-Proto: https     │
                        └──────────────────┬──────────────────────────┘
                                           │ HTTP
                        ┌──────────────────▼──────────────────────────┐
                        │ docker compose stack (showcase)             │
                        │                                             │
                        │  nginx (контейнер) :80                      │
                        │   ├── /assets/, /  → SPA (frontend-dist)    │
                        │   └── /api,/oauth2,/login,... → backend:8080│
                        │                                             │
                        │  backend (Spring Boot) :8080                │
                        │   ├── Postgres ─► postgres:5432             │
                        │   └── S3       ─► minio:9000                │
                        │                                             │
                        │  postgres :5432   minio :9000               │
                        └─────────────────────────────────────────────┘
```

## Что нужно поднять на машине

1. **DNS**: A-запись домена (например `showcase.example.com`) → публичный IP VM.
2. **Docker + Compose v2**, `git`, `nginx` (host-уровневый), `certbot`.
3. **Открытые порты**: `80` (для ACME-челленджа и редиректа), `443` (HTTPS).
   Порт `8081` стека наружу **не выставляем** — он на `127.0.0.1`.
4. Клон репозитория + `.env`:
   ```bash
   git clone --recurse-submodules <repo-url>
   cd showcase
   cp .env.example .env
   # FRONT_URL=https://showcase.example.com
   # APP_DEPLOYMENT_MODE=prod  (или dev, если без Azure)
   # AZURE_TENANT_ID / AZURE_CLIENT_ID / AZURE_CLIENT_SECRET (для prod)
   docker compose up -d --build
   ```
5. **TLS-сертификат**:
   ```bash
   sudo certbot --nginx -d showcase.example.com
   ```

## host nginx (терминация HTTPS → HTTP в контейнер)

Создаём `/etc/nginx/sites-available/showcase`:

```nginx
# HTTP → HTTPS
server {
    listen 80;
    server_name showcase.example.com;
    return 301 https://$host$request_uri;
}

# HTTPS → стек на 127.0.0.1:8081
server {
    listen 443 ssl http2;
    server_name showcase.example.com;

    ssl_certificate     /etc/letsencrypt/live/showcase.example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/showcase.example.com/privkey.pem;

    client_max_body_size 64m;

    location / {
        proxy_pass http://127.0.0.1:8081;
        proxy_http_version 1.1;
        proxy_set_header Host              $host;
        proxy_set_header X-Real-IP         $remote_addr;
        proxy_set_header X-Forwarded-For   $proxy_add_x_forwarded_for;
        # Критично: Spring строит OAuth redirect_uri из этих заголовков.
        # SameSite=None; Secure cookie не сядет, если Proto != https.
        proxy_set_header X-Forwarded-Proto https;
        proxy_set_header X-Forwarded-Host  $host;
        proxy_set_header X-Forwarded-Port  443;
        proxy_read_timeout 120s;
    }
}
```

```bash
sudo ln -s /etc/nginx/sites-available/showcase /etc/nginx/sites-enabled/
sudo nginx -t && sudo systemctl reload nginx
```

## Почему именно так

- **Двойной nginx** (host + контейнер) — host отвечает за TLS и сертификаты
  (certbot обновляет их вне Docker), контейнерный nginx раздаёт SPA и
  маршрутизирует на backend в своей сети.
- **`X-Forwarded-Proto: https`** обязателен: Spring c
  `server.forward-headers-strategy=framework` использует его для построения
  абсолютных OAuth `redirect_uri`. Без него Azure вернёт ошибку
  redirect_uri_mismatch, а cookie сессии (`SameSite=None; Secure`) не установится.
- **Стек слушает `127.0.0.1`**: исключает прямой доступ к HTTP в обход TLS.
- **Azure App Registration** redirect URI должен быть
  `https://<домен>/login/oauth2/code/azure` — см. [DEPLOY.md](DEPLOY.md).
