
# 📘 Plyglo Learning Platform — Architecture Specification

```text
██████╗  ██╗    ██╗   ██╗  ██████╗  ██╗       ██████╗
██╔══██╗ ██║    ╚██╗ ██╔╝ ██╔════╝  ██║      ██╔═══██╗
██████╔╝ ██║     ╚████╔╝  ██║  ███╗ ██║      ██║   ██║
██╔═══╝  ██║      ╚██╔╝   ██║   ██║ ██║      ██║   ██║
██║      ███████╗  ██║    ╚██████╔╝ ███████╗ ╚██████╔╝
╚═╝      ╚══════╝  ╚═╝     ╚═════╝  ╚══════╝  ╚═════╝

                 .md  ⇄  A  ⇄  文

                    plyglo.com
```

## 1. Контекст проекта

**Plyglo** — образовательная платформа. Состоит из нескольких фронтендов и набора микросервисов, объединённых через reverse proxy (Caddy). Аутентификация — через self-hosted Authentik (OIDC). Авторизация — на старте простая (сессия + enrollment), с возможностью введения OPA позже.

**Инфраструктура:** мини-сервер на Proxmox. Несколько VM. Docker + Docker Compose на каждой VM.

**Домены:**
- `plyglo.com` — основной сайт (Nuxt, SSR + фронтенд)
- `study.plyglo.com` — приложение для обучения (Expo web)
- `api.plyglo.com` — **тот же Nuxt-контейнер**, что и `plyglo.com` (BFF + API)
- `auth.plyglo.com` — **публичный** алиас Authentik (IdP)
- `grafana.plyglo.com` — опционально, дашборды (с аутентификацией)

---

## 2. Ключевые архитектурные решения

1. **Reverse proxy — Caddy (Решение №4).** Заменяет Traefik. Декларативный Caddyfile в Git, автоматический TLS, ForwardAuth через `forward_auth`.
2. **BFF — это Nuxt (Nitro), один контейнер на два домена (Решение №2, уточнено).** `plyglo.com` и `api.plyglo.com` — это **один и тот же процесс Nuxt**. Caddy роутит оба домена на `nuxt:3000`. SSR-страницы живут в `app/pages/*`, BFF-роуты — в `server/routes/*`. Токены не попадают в браузер.
3. **`auth.plyglo.com` — публичный алиас Authentik (Решение №1).** OIDC issuer — публичный URL. Браузер ходит на `auth.plyglo.com` напрямую. Требует обязательной защиты.
4. **CSRF — `SameSite=Lax` + CSRF-токен, доставляемый через `/me` (Решение №3, уточнено).** Токен хранится в Redis (`csrf:{session_id}`), но **не** в куке. BFF возвращает его в теле ответа `/me`. Фронт хранит токен в памяти и шлёт в заголовке `X-CSRF-Token`.
5. **CORS обязателен.** `plyglo.com` и `api.plyglo.com` — разные origin. BFF разрешает cross-origin запросы от `https://plyglo.com` и `https://study.plyglo.com` с `Access-Control-Allow-Credentials: true`.
6. **OPA откладывается (Решение №5).** На старте RBAC реализуется проще (см. раздел 7).
7. **Модель доступа:**
   - **Публично:** `GET /api/courses`, `GET /api/courses/{id}`, `GET /api/courses/{id}/chapters`.
   - **Аутентифицированно:** `/api/me`, `/api/enrollments`, `/api/progress/*`, `/api/courses/{id}/lessons*`.
   - **Enrollment:** self-enrollment через `POST /api/enrollments`.
   - **Admin:** `/api/admin/*` — только роль `admin`.
8. **Роли:**
   - **Глобальные** (`student`, `admin`) — в Authentik (группы), попадают в `id_token` как claim `groups`.
   - **Внутри курса** (`student` / `assistant` / `teacher`) — в `enrollment.role` в БД микросервиса `courses`.
9. **Микросервисы — на Go.** `courses`, `progress`, `users` (`chi` + `net/http` + `prometheus/client_golang` + `log/slog`).
10. **Мониторинг — на отдельной VM.**
11. **API-версионирование — через URL-префиксы, не через отдельные VM.** Вводится только когда появится внешний клиент.
12. **Разделение по VM — по функциональным зонам.**

---

## 3. Топология VM

```
┌──────────────────────────────────────────────────────────────────────────────┐
│                          PROXMOX HOST (mini-server)                          │
│                                                                              │
│  vmbr0 (external, internet)  │  vmbr1 (internal, no internet)                │
│  ─────────────────────────────┼─────────────────────────────────────────    │
│                               │                                              │
│  ┌────────────────────────────┴──────────────────────────────────────────┐   │
│  │ VM #1 — EDGE                                                          │   │
│  │   • Caddy (публичный, 80/443, TLS)                                    │   │
│  │   • Nuxt (plyglo.com + api.plyglo.com) — SSR + BFF (Nitro)            │   │
│  │   • study.plyglo.com (Expo web, static)                               │   │
│  │   • Агенты: node-exporter, cAdvisor, promtail                         │   │
│  └───────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
│  ┌───────────────────────────────────────────────────────────────────────┐   │
│  │ VM #2 — AUTH                                                          │   │
│  │   • Authentik (server + worker)                                       │   │
│  │   • PostgreSQL (для Authentik)                                        │   │
│  │   • Redis (сессии BFF + CSRF + кэш Authentik)                         │   │
│  │   • Агенты: node-exporter, cAdvisor, promtail                         │   │
│  │   (OPA — отложен)                                                     │   │
│  └───────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
│  ┌───────────────────────────────────────────────────────────────────────┐   │
│  │ VM #3 — BUSINESS                                                      │   │
│  │   • courses (Go)                                                      │   │
│  │   • progress (Go)                                                     │   │
│  │   • users (Go)                                                        │   │
│  │   • PostgreSQL (БД сервисов)                                          │   │
│  │   • Агенты: node-exporter, cAdvisor, promtail                         │   │
│  └───────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
│  ┌───────────────────────────────────────────────────────────────────────┐   │
│  │ VM #4 — MONITORING                                                    │   │
│  │   • Prometheus, Grafana, Alertmanager, Loki                           │   │
│  │   • Агенты: node-exporter, cAdvisor, promtail                         │   │
│  └───────────────────────────────────────────────────────────────────────┘   │
└──────────────────────────────────────────────────────────────────────────────┘
```

**Сеть:**
- `vmbr0` — внешний bridge, выход в интернет, публичный IP.
- `vmbr1` — внутренний bridge без физического интерфейса. Только VM между собой.
- Публично слушает **только VM #1** (Caddy, 80/443).
- Внутренние DNS-имена: `authentik.internal.lan`, `redis.internal.lan`, `courses.internal.lan` и т.д.

---

## 4. Роли компонентов

| Компонент | Роль | Что хранит | Что НЕ делает |
|---|---|---|---|
| **Caddy (VM #1)** | Entry point, TLS, маршрутизация, ForwardAuth | Caddyfile (в Git) | Не принимает решения о доступе |
| **Nuxt / Nitro (VM #1)** | BFF: OIDC-клиент, сессии, CSRF, проксирование API. Обслуживает `plyglo.com` (SSR) и `api.plyglo.com` (BFF) | Session ID → user_id, refresh_token (в Redis) | Не применяет RBAC |
| **Authentik (VM #2)** | Identity Provider (OIDC) | Пользователи, группы, MFA | Не знает про микросервисы Plyglo |
| **Redis (VM #2)** | Сессии + CSRF-токены + кэш | session → {user_id, roles, refresh_token} | — |
| **Микросервисы (VM #3, Go)** | Бизнес-логика + enrollment + тонкая авторизация | Свои данные | Не аутентифицируют |
| **Prometheus/Grafana/Loki (VM #4)** | Observability | Метрики, логи, алерты | — |

---

## 5. Поток аутентификации (AuthN)

**Компоненты:** браузер, Caddy, Nuxt/Nitro (BFF), Authentik.

**Шаги:**

1. Пользователь открывает `plyglo.com`. Фронт делает `GET https://api.plyglo.com/me` с `credentials: 'include'`.
2. Нет валидной сессии → `401`. Фронт редиректит браузер на `https://api.plyglo.com/auth/login`.
3. **Nuxt BFF** генерирует `state` + `code_verifier` (PKCE), кладёт во временную куку, формирует authorization URL для Authentik:
   ```
   https://auth.plyglo.com/application/o/plyglo-api/authorize/
     ?client_id=...
     &redirect_uri=https://api.plyglo.com/auth/callback
     &response_type=code
     &scope=openid profile email
     &state=...
     &code_challenge=...
     &code_challenge_method=S256
   ```
   Делает `302`.
4. Браузер идёт **напрямую** на `https://auth.plyglo.com/...`. Caddy на VM #1 применяет цепочку `[security-headers] → [ratelimit-strict] → [block-metrics]`, проксирует на Authentik (VM #2).
5. Пользователь видит форму входа, вводит данные.
6. Authentik редиректит браузер на `https://api.plyglo.com/auth/callback?code=...&state=...`.
7. **Nuxt BFF**:
   - Проверяет `state` из временной куки.
   - Делает `POST` на `https://auth.plyglo.com/application/o/token/` с `client_id` + `client_secret` + `code` + `redirect_uri` + `code_verifier`.
   - Получает `id_token`, `access_token`, `refresh_token`.
   - Валидирует `id_token` через JWKS.
   - Извлекает `sub`, `email`, `name`, `groups`.
   - Генерирует session ID (`crypto.randomBytes(32).toString('base64url')`), сохраняет в Redis:
     ```
     session:{id} → {
       user_id, email, name, roles,
       access_token (encrypted), refresh_token (encrypted),
       access_exp, created_at, last_seen,
       user_agent, ip
     }
     csrf:{id} → "<csrf_token>"
     user_sessions:{user_id} → SET { session_id, ... }
     ```
   - Ставит **только одну куку** `__Host-session` (`HttpOnly`, `Secure`, `SameSite=Lax`).
8. Редирект браузера на исходный фронт.
9. Фронт снова делает `GET /me` с кукой. BFF:
   - Валидирует сессию в Redis.
   - Читает `csrf:{session_id}`.
   - Возвращает JSON:
     ```json
     {
       "user": { "id": "usr_42", "email": "user@plyglo.com", "name": "Иван" },
       "csrf_token": "abc123..."
     }
     ```
10. Фронт сохраняет `csrf_token` в памяти (Pinia store / composable). Дальнейшие небезопасные запросы шлёт с `X-CSRF-Token: abc123...`.

**Logout:**
1. Фронт вызывает `POST https://api.plyglo.com/auth/logout` с `X-CSRF-Token`.
2. BFF: удаляет сессию из Redis, отзывает `refresh_token` в Authentik, чистит куку.

---

## 6. CSRF-защита (уточнено)

**Стратегия:** `SameSite=Lax` + CSRF-токен через тело ответа `/me`.

**Почему не через куку:**
- `plyglo.com` и `api.plyglo.com` — разные origin.
- Кука, установленная на `api.plyglo.com`, **не видна** из JS на `plyglo.com` (`document.cookie` показывает только куки текущего домена).
- Установка куки с `Domain=.plyglo.com` ослабила бы защиту (`__Host-` префикс невозможен, кука уходит на все поддомены).

**Как работает:**
1. При логине BFF генерирует `csrf_token` (`crypto.randomBytes(32).toString('base64url')`), сохраняет в Redis: `csrf:{session_id} → "<token>"`, TTL = TTL сессии.
2. BFF **не** кладёт токен в куку.
3. При каждом `GET /me` BFF возвращает `csrf_token` в JSON.
4. Фронт хранит токен **в памяти** (Pinia store или composable). **Не** в localStorage, **не** в sessionStorage.
5. При небезопасных запросах (POST/PUT/DELETE/PATCH) фронт шлёт `X-CSRF-Token: <token>`.
6. BFF сверяет заголовок с `csrf:{session_id}` в Redis через `crypto.timingSafeEqual`.
7. При перезагрузке страницы фронт заново дёргает `/me` — токен восстанавливается (это и так происходит для профиля).

**Дополнительно:**
- BFF валидирует `Origin`/`Referer` для небезопасных методов (defense in depth).
- GET/HEAD/OPTIONS не требуют токена.

**Пример middleware в Nuxt:**

```typescript
// server/middleware/csrf.ts
export default defineEventHandler(async (event) => {
  const method = event.method
  if (['GET', 'HEAD', 'OPTIONS'].includes(method)) return

  const sessionId = getCookie(event, '__Host-session')
  if (!sessionId) throw createError({ statusCode: 401 })

  const storedToken = await redis.get(`csrf:${sessionId}`)
  const headerToken = getHeader(event, 'x-csrf-token')

  if (!storedToken || !headerToken) {
    throw createError({ statusCode: 403, statusMessage: 'CSRF token missing' })
  }

  const a = Buffer.from(storedToken)
  const b = Buffer.from(headerToken)
  if (a.length !== b.length || !timingSafeEqual(a, b)) {
    throw createError({ statusCode: 403, statusMessage: 'CSRF token mismatch' })
  }

  // Дополнительно: валидация Origin
  const origin = getHeader(event, 'origin')
  const allowed = ['https://plyglo.com', 'https://study.plyglo.com']
  if (origin && !allowed.includes(origin)) {
    throw createError({ statusCode: 403, statusMessage: 'Origin not allowed' })
  }
})
```

---

## 7. CORS (новое)

**Проблема:** `plyglo.com`, `study.plyglo.com` и `api.plyglo.com` — три разных origin. Браузер применяет CORS-политику к каждому cross-origin запросу.

**Что нужно настроить на Nuxt BFF:**

1. **Разрешить origins:**
   ```typescript
   // nuxt.config.ts или server/middleware/cors.ts
   const allowedOrigins = [
     'https://plyglo.com',
     'https://study.plyglo.com',
     'https://www.plyglo.com',  // если есть
   ]
   ```

2. **Заголовки для preflight (`OPTIONS`) и обычных запросов:**
   ```
   Access-Control-Allow-Origin: https://plyglo.com   (эхо реального origin)
   Access-Control-Allow-Credentials: true
   Access-Control-Allow-Methods: GET, POST, PUT, PATCH, DELETE, OPTIONS
   Access-Control-Allow-Headers: Content-Type, X-CSRF-Token
   Access-Control-Max-Age: 86400
   ```

3. **Важно:** нельзя использовать `Access-Control-Allow-Origin: *` вместе с `Access-Control-Allow-Credentials: true`. Нужно **эхо** конкретного origin из белого списка.

**Пример middleware:**

```typescript
// server/middleware/cors.ts
const allowedOrigins = [
  'https://plyglo.com',
  'https://study.plyglo.com',
]

export default defineEventHandler((event) => {
  const origin = getHeader(event, 'origin')

  if (origin && allowedOrigins.includes(origin)) {
    setHeader(event, 'Access-Control-Allow-Origin', origin)
    setHeader(event, 'Access-Control-Allow-Credentials', 'true')
    setHeader(event, 'Vary', 'Origin')
  }

  if (event.method === 'OPTIONS') {
    setHeader(event, 'Access-Control-Allow-Methods', 'GET, POST, PUT, PATCH, DELETE, OPTIONS')
    setHeader(event, 'Access-Control-Allow-Headers', 'Content-Type, X-CSRF-Token')
    setHeader(event, 'Access-Control-Max-Age', '86400')
    return ''
  }
})
```

**Альтернатива:** проксировать API через тот же origin. Например, `plyglo.com/api/*` → Nuxt BFF. Тогда фронт ходит на `plyglo.com/api/*` (same-origin), CORS не нужен, CSRF можно вернуть в куку. Но `api.plyglo.com` всё равно остаётся публичным для внешних клиентов.

**Решение для Plyglo:** оставляем cross-origin между фронтом и API, настраиваем CORS, используем CSRF-токен через `/me`.

---

## 8. Поток авторизации (AuthZ) — упрощённый (без OPA)

**Модель доступа:**

| Эндпоинт | Доступ | Проверка |
|---|---|---|
| `GET /api/courses` | Публично | Никакой |
| `GET /api/courses/{id}` | Публично | Никакой |
| `GET /api/courses/{id}/chapters` | Публично | Никакой |
| `GET /api/courses/{id}/lessons` | Только записанные | Сессия + enrollment |
| `GET /api/courses/{id}/lessons/{lid}` | Только записанные | Сессия + enrollment |
| `POST /api/enrollments` | Аутентифицированные | Сессия |
| `GET /api/progress/me` | Аутентифицированные | Сессия |
| `POST /api/progress` | Записанные | Сессия + enrollment |
| `POST /api/courses/{id}/lessons` | teacher курса | Сессия + enrollment.role |
| `GET /api/admin/*` | admin | Сессия + роль `admin` |

**Цепочка middleware в Caddy:**

```
Публичные роуты:      [security-headers] → Nuxt
Приватные роуты:      [security-headers] → [forward_auth] → Nuxt
auth.plyglo.com:      [security-headers] → [ratelimit-strict] → [block-metrics] → Authentik
```

**Двухуровневая модель:**
- **Уровень 1 (Caddy + Nuxt BFF):** «аноним / аутентифицирован».
- **Уровень 2 (микросервис `courses`):** «записан / не записан на курс».
- **Роль `admin`:** проверяется в Nuxt BFF.

**Поток для `GET /api/courses/123/lessons`:**

1. Браузер → Caddy: `GET https://api.plyglo.com/api/courses/123/lessons` + кука.
2. **Caddy forward_auth:** проверяет сессию через Nuxt BFF (`/auth/verify`). Нет сессии → `302` на `/auth/login`.
3. Caddy → Nuxt BFF с заголовками от Authentik Outpost.
4. Nuxt BFF проксирует на микросервис `courses` (VM #3), добавляя `X-User-ID`, `X-User-Roles`.
5. Микросервис `courses` проверяет enrollment `(user_id=42, course_id=123)`.
   - Есть → отдаёт уроки.
   - Нет → `403 Forbidden`.

**Поток для `GET /api/courses` (публичный):**

1. Браузер → Caddy: `GET https://api.plyglo.com/api/courses`.
2. Caddy: без forward_auth, сразу проксирует на Nuxt BFF.
3. Nuxt BFF проксирует на микросервис `courses`.
4. Микросервис отдаёт список курсов.

**Когда вводить OPA:** когда появятся сложные правила.

---

## 9. Безопасность публичного Authentik

**1. Rate limiting (Caddy):**
```caddy
auth.plyglo.com {
    rate_limit {
        zone auth_zone {
            key {remote_host}
            events 10
            window 1s
        }
    }
}
```

**2. Security headers:**
```caddy
header {
    Strict-Transport-Security "max-age=31536000; includeSubDomains; preload"
    X-Content-Type-Options "nosniff"
    X-Frame-Options "DENY"
    Referrer-Policy "strict-origin-when-cross-origin"
}
```

**3. Блокировка `/-/metrics/` публично.**

**4. IP allowlist для `/if/admin/`** (опционально).

**5. Обязательные параметры Authentik:**
```yaml
AUTHENTIK_HOST: "https://auth.plyglo.com"
AUTHENTIK_HOST_BROWSER: "https://auth.plyglo.com"
AUTHENTIK_SECRET_KEY: "<openssl rand -base64 60 | tr -d '\n'>"
AUTHENTIK_LISTEN__TRUSTED_PROXY_CIDRS: "10.10.1.0/24,10.10.2.0/24"
AUTHENTIK_LISTEN__HTTPS: ""
```

**6. Мониторинг аномалий:**
- Алерт на `authentik_login_failed_total` rate > 10/min.
- Алерт на `authentik_http_requests_total{status="5xx"}` rate > 1%.

**7. Обновления:** 24–48 часов после security-патча.

---

## 10. Формат сессий в Redis

```
session:{session_id}              (TTL: 7 дней, sliding)
├── user_id        "usr_abc123"
├── email          "user@plyglo.com"
├── name           "Иван Иванов"
├── roles          "student,beta-tester"
├── access_token   "<encrypted>"
├── refresh_token  "<encrypted>"
├── access_exp     "1735689600"
├── created_at     "1735084800"
├── last_seen      "1735603200"
├── user_agent     "Mozilla/5.0..."
└── ip             "203.0.113.42"

user_sessions:{user_id}           (TTL: 90 дней)
└── SET { session_id_1, session_id_2, ... }

csrf:{session_id}                 (TTL: 7 дней, синхронно с session)
└── "<csrf_token>"
```

**Параметры:**
- Session TTL: **30 дней** (абсолютный максимум).
- Idle timeout: **7 дней**.
- Sliding expiration: обновлять TTL, если `last_seen` старше 5 минут.
- Access token: **15 минут**.
- Refresh token: **30 дней**.
- Rotation refresh_token: **включён**.
- Шифрование refresh_token: **AES-GCM**, ключ в env Nuxt.
- Session ID: `crypto.randomBytes(32).toString('base64url')`.

---

## 11. Мониторинг и observability

**Центральный стек (VM #4):**
- **Prometheus** — метрики (retention 15–30 дней).
- **Grafana** — дашборды.
- **Alertmanager** — алерты (Telegram/email).
- **Loki** — логи (retention 7–14 дней).

**Агенты (на каждой VM):** `node-exporter`, `cAdvisor`, `promtail`.

**Что мониторим:**

| Источник | Ключевые метрики |
|---|---|
| Caddy | RPS, latency p50/p95/p99, коды ответов |
| Authentik | Логины, ошибки, время ответа token-эндпоинта |
| Nuxt BFF | RED-метрики, ошибки OIDC, время token-обмена |
| Микросервисы (Go) | RED-метрики по каждому |
| PostgreSQL | Активные соединения, медленные запросы |
| Redis | Memory usage, hit/miss, количество ключей |
| VM | CPU, RAM, диск, load average |

**Алерты (минимум):**
- 5xx rate > 5%.
- Latency p95 > 1s.
- Authentik недоступен > 30s.
- Диск > 80%.
- Redis memory > 80%.
- `authentik_login_failed_total` rate > 10/min.

---

## 12. Разделение ответственности

| Слой | VM | Что там |
|---|---|---|
| **Edge** | VM #1 | Caddy, Nuxt (SSR + BFF), Expo web |
| **Auth** | VM #2 | Authentik, PostgreSQL, Redis |
| **Business** | VM #3 | courses, progress, users (Go), PostgreSQL |
| **Monitoring** | VM #4 | Prometheus, Grafana, Alertmanager, Loki |

**Принципы:**
- Публично слушает **только VM #1**.
- VM #2, #3, #4 доступны только по `vmbr1`.
- Микросервисы не аутентифицируют — доверяют заголовкам от Nuxt BFF.
- Authentik не знает про микросервисы.
- Nuxt BFF — единственный, кто ходит в Authentik по OIDC и в Redis за сессиями.

---

## 13. Структура Nuxt-приложения

```
nuxt-app/
├── app/
│   ├── pages/                  ← SSR-страницы (plyglo.com)
│   │   ├── index.vue
│   │   ├── courses/
│   │   └── about.vue
│   ├── components/
│   └── composables/
│       └── useAuth.ts          ← хранит csrf_token в памяти, шлёт с запросами
│
├── server/
│   ├── routes/                 ← BFF-роуты (api.plyglo.com)
│   │   ├── auth/
│   │   │   ├── login.get.ts
│   │   │   ├── callback.get.ts
│   │   │   ├── logout.post.ts
│   │   │   └── verify.get.ts   ← для Caddy forward_auth
│   │   ├── me.get.ts           ← возвращает user + csrf_token
│   │   └── api/
│   │       └── [...path].ts    ← прокси на VM#3
│   │
│   ├── middleware/
│   │   ├── session.ts          ← валидация сессии
│   │   ├── csrf.ts             ← проверка X-CSRF-Token
│   │   └── cors.ts             ← CORS для plyglo.com / study.plyglo.com
│   │
│   └── utils/
│       ├── oidc.ts
│       ├── redis.ts
│       └── crypto.ts
│
└── nuxt.config.ts
```

---

## 14. Текущее состояние и открытые вопросы

**Уже сделано / решено:**
- Proxmox + несколько VM.
- Reverse proxy — Caddy (Решение №4).
- BFF — Nuxt (Nitro), Go — для микросервисов (Решение №2).
- `api.plyglo.com` — тот же Nuxt-контейнер, что и `plyglo.com` (уточнено).
- `auth.plyglo.com` — публичный алиас Authentik (Решение №1).
- CSRF: токен через `/me`, не через куку (Решение №3, уточнено).
- CORS обязателен между `plyglo.com` и `api.plyglo.com` (уточнено).
- OPA отложен (Решение №5).
- Формат сессий в Redis зафиксирован (Решение №6).
- Модель доступа: публичные курсы + enrollment.

**Открытые вопросы на следующий раз:**
1. **Внутренний DNS на `vmbr1`:** Pi-hole? dnsmasq? `/etc/hosts`?
2. **Структура API:** полный список эндпоинтов.
3. **TLS для внутреннего трафика между VM:** mTLS?
4. **Бэкапы:** что, как часто, куда.
5. **Порядок развёртывания** VM и сервисов.
6. **Конкретный `docker-compose.yml`** для каждой VM.
7. **Конкретный Caddyfile** для VM #1.
8. **Scrape-конфиг Prometheus**.
9. **Настройка Proxmox bridge `vmbr1`**.

---

## 15. Стек технологий

- **Гипервизор:** Proxmox VE
- **ОС на VM:** Linux (Debian/Ubuntu)
- **Контейнеризация:** Docker + Docker Compose
- **Reverse proxy:** Caddy
- **IdP:** Authentik (OIDC, self-hosted)
- **БД для Authentik:** PostgreSQL
- **Кэш/сессии:** Redis
- **Фронтенды:** Nuxt (plyglo.com + api.plyglo.com, SSR + BFF), Expo web (study.plyglo.com)
- **BFF:** Nuxt Nitro (TypeScript), OIDC через `openid-client`
- **Микросервисы:** Go (`chi` + `net/http` + `prometheus/client_golang` + `log/slog`)
- **Мониторинг:** Prometheus, Grafana, Alertmanager, Loki, node-exporter, cAdvisor, promtail

---

## 🖥️ Детальные схемы VM

### VM #1 — EDGE (Caddy + Nuxt BFF)

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                              VM #1 — EDGE                                           │
│                          (единственная публичная VM)                                │
│                                                                                     │
│  vmbr0 (публичный IP, 80/443)          vmbr1 (внутренний, 10.10.1.10)               │
│  ────────────────┬─────────────────────┬────────────────────────────────────────    │
│                  │                     │                                            │
│                  ▼                     │                                            │
│  ┌──────────────────────────────────────────────────────────────┐                   │
│  │                       CADDY                                  │                   │
│  │  • :80  → redirect to :443                                   │                   │
│  │  • :443 → TLS termination (Let's Encrypt, ACME)              │                   │
│  │  • :2019 → admin API (internal)                              │                   │
│  │  • :9180 → metrics (Prometheus)                              │                   │
│  │                                                              │                   │
│  │  Caddyfile:                                                  │                   │
│  │   plyglo.com/*       → nuxt:3000                             │                   │
│  │   study.plyglo.com/* → expo:80                               │                   │
│  │   api.plyglo.com/*   → nuxt:3000  (тот же контейнер!)        │                   │
│  │   auth.plyglo.com/*  → 10.10.2.12:9000  (Authentik)          │                   │
│  │                                                              │                   │
│  │  Middleware chains:                                          │                   │
│  │   api.plyglo.com (публичные):  [security-headers] → nuxt     │                   │
│  │   api.plyglo.com (приватные):  [security-headers]            │                   │
│  │                                → [forward_auth] → nuxt       │                   │
│  │   auth.plyglo.com:  [security-headers]                       │                   │
│  │                     → [ratelimit-strict]                     │                   │
│  │                     → [block-metrics] → Authentik            │                   │
│  └──────────────────────────────────────────────────────────────┘                   │
│                  │                                                                  │
│                  ▼                                                                  │
│  ┌────────────────────────────────┐  ┌────────────────────┐                         │
│  │  Nuxt (ОДИН КОНТЕЙНЕР)         │  │ study.plyglo.com   │                         │
│  │  • :3000                       │  │ (Expo web, nginx)  │                         │
│  │  • plyglo.com  → SSR pages     │  │ • :80              │                         │
│  │  • api.plyglo.com → BFF        │  │                    │                         │
│  │                                │  │                    │                         │
│  │  Server routes (BFF):          │  │                    │                         │
│  │   /auth/login                  │  │                    │                         │
│  │   /auth/callback               │  │                    │                         │
│  │   /auth/logout                 │  │                    │                         │
│  │   /auth/verify  (для Caddy)    │  │                    │                         │
│  │   /me  → user + csrf_token     │  │                    │                         │
│  │   /api/*  → proxy to VM#3      │  │                    │                         │
│  │                                │  │                    │                         │
│  │  Middleware:                   │  │                    │                         │
│  │   session, csrf, cors          │  │                    │                         │
│  │                                │  │                    │                         │
│  │  Ходит в:                      │  │                    │                         │
│  │   → auth.plyglo.com (OIDC)     │  │                    │                         │
│  │   → Redis (сессии)             │  │                    │                         │
│  │   → VM#3 (прокси API)          │  │                    │                         │
│  └────────────────────────────────┘  └────────────────────┘                         │
│                                                                                     │
│  ┌──────────────────────────────────────────────────────────────┐                   │
│  │  AGENTS: node-exporter :9100, cAdvisor :8080, promtail       │                   │
│  └──────────────────────────────────────────────────────────────┘                   │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

**Ключевое:**
- Caddy роутит `plyglo.com` и `api.plyglo.com` на **один и тот же** контейнер `nuxt:3000`.
- Nuxt сам разбирается: SSR-страницы для `plyglo.com`, BFF-роуты для `api.plyglo.com`.
- `csrf_token` возвращается в теле `/me`, не в куке.
- CORS-заголовки добавляются middleware `cors.ts` для `plyglo.com` и `study.plyglo.com`.

---

### VM #2 — AUTH

*(без изменений по сравнению с предыдущей версией — добавлен Redis для CSRF)*

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                            VM #2 — AUTH                                             │
│                       (внутренняя, 10.10.2.0/24)                                    │
│                                                                                     │
│  vmbr1 (10.10.2.10)                                                                 │
│  ────────────────┬─────────────────────────────────────────────────────────────     │
│                  │                                                                  │
│                  ▼                                                                  │
│  ┌────────────────────────────────────────────────────────────┐                     │
│  │   AUTHENTIK (публично через auth.plyglo.com)               │                     │
│  │  • server :9000, worker, :9443                             │                     │
│  │  Endpoints: /application/o/*, /if/admin/, /-/metrics/      │                     │
│  └──────────┬─────────────────────────────────────────────────┘                     │
│             │                                                                       │
│             ▼                                                                       │
│  ┌──────────────────┐                                                               │
│  │   POSTGRESQL     │                                                               │
│  │  • :5432         │                                                               │
│  └──────────────────┘                                                               │
│                                                                                     │
│  ┌──────────────────┐                                                               │
│  │     REDIS        │                                                               │
│  │  • :6379         │                                                               │
│  │  session:{id}    │                                                               │
│  │  csrf:{id}       │                                                               │
│  │  user_sessions:  │                                                               │
│  │   {user_id}      │                                                               │
│  └──────────────────┘                                                               │
│                                                                                     │
│  ┌──────────────────────────────────────────────────────────────────────────────┐   │
│  │  AGENTS: node-exporter :9100, cAdvisor :8080, promtail                       │   │
│  └──────────────────────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

---

### VM #3 — BUSINESS

*(без изменений — courses, progress, users на Go + PostgreSQL)*

---

### VM #4 — MONITORING

*(без изменений)*

---

## 🔗 Сводная карта связей между VM

```
                          ┌─────────────────────────────┐
                          │        ИНТЕРНЕТ             │
                          └──────────────┬──────────────┘
                                         │ :80/:443
                                         ▼
┌───────────────────────────────────────────────────────────────────────────────────┐
│  VM#1 EDGE (10.10.1.10)                                                           │
│   Caddy ────┬──► nuxt:3000 (plyglo.com SSR)                                       │
│             ├──► nuxt:3000 (api.plyglo.com BFF)                                   │
│             ├──► expo:80   (study.plyglo.com)                                     │
│             ├──► 10.10.2.12:9000  Authentik (auth.plyglo.com)                     │
│             └──► 10.10.2.11:6379  Redis (session check)                           │
│                                                                                   │
│   Nuxt BFF ─┬──► auth.plyglo.com  (OIDC: authorize, token, jwks)                  │
│             ├──► 10.10.2.11:6379  Redis (сессии + CSRF)                           │
│             └──► 10.10.3.x        Микросервисы (прокси API)                       │
└───────────────────────────────────────────────────────────────────────────────────┘
                                         │
        ┌────────────────────────────────┼────────────────────────────────┐
        ▼                                ▼                                ▼
┌───────────────────────┐   ┌────────────────────────┐   ┌──────────────────────────┐
│ VM#2 AUTH (10.10.2.x) │   │ VM#3 BUSINESS (10.10.3)│   │ VM#4 MONITOR (10.10.4)   │
│  Authentik, PG, Redis │   │  courses, progress,    │   │  Prometheus, Grafana,    │
│                       │◄──│  users (Go), PG        │   │  Loki, Alertmanager      │
└───────────────────────┘   └────────────────────────┘   └──────────────────────────┘
```

---

## 📋 Сводная таблица портов

| VM | Сервис | Порт | Доступен из |
|---|---|---|---|
| VM#1 | Caddy (http) | 80 | Интернет |
| VM#1 | Caddy (https) | 443 | Интернет |
| VM#1 | Caddy metrics | 9180 | VM#4 (Prometheus) |
| VM#1 | Nuxt (SSR + BFF) | 3000 | Caddy (local) |
| VM#1 | Expo web | 80 | Caddy (local) |
| VM#2 | Authentik | 9000 | Caddy (VM#1), Nuxt BFF |
| VM#2 | Authentik (TLS) | 9443 | Nuxt BFF (VM#1) |
| VM#2 | Redis | 6379 | VM#1 (Caddy + Nuxt BFF) |
| VM#2 | PostgreSQL (Authentik) | 5432 | Authentik |
| VM#3 | courses | 8001 | Nuxt BFF (VM#1) |
| VM#3 | progress | 8002 | Nuxt BFF (VM#1) |
| VM#3 | users | 8003 | Nuxt BFF (VM#1) |
| VM#3 | PostgreSQL (сервисы) | 5432 | Микросервисы |
| VM#4 | Prometheus | 9090 | Grafana |
| VM#4 | Grafana | 3000 | VM#1 (Caddy, опционально) |
| VM#4 | Loki | 3100 | promtail со всех VM |
| VM#4 | Alertmanager | 9093 | Prometheus |
| Все | node-exporter | 9100 | VM#4 (Prometheus) |
| Все | cAdvisor | 8080 | VM#4 (Prometheus) |

