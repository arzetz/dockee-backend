# dockee-backend

Серверная часть системы автоматизированного анализа документов **Dockee**.  
Выявляет юридические и финансовые риски в документах МСП с помощью ИИ-агента и встроенной NER-анонимизации персональных данных.

---

## Стек

| Компонент | Технология |
|---|---|
| Язык | Go 1.24 |
| HTTP-фреймворк | Gin |
| СУБД | PostgreSQL 17 (JSONB, RBAC) |
| Файловое хранилище | MinIO (S3-совместимый интерфейс) |
| Миграции БД | golang-migrate |
| Аутентификация | JWT (RFC 7519), bcrypt cost 12 |
| ИИ-сервис | DeepSeek API (адаптер ai_connector.go) |
| Анонимизация ПДн | NER на регулярных выражениях (ner.go) |
| API-документация | OpenAPI 3.0, swaggo |
| Контейнеризация | Docker, Docker Compose |
| CI | GitHub Actions |

---

## Архитектура

Модульный монолит с трёхслойной чистой архитектурой:

```
Transport Layer   →   Service Layer   →   Repository Layer
 (Gin, middleware)    (бизнес-логика)     (PostgreSQL, S3)
```

Анонимизация персональных данных встроена в конвейер обработки документа как **обязательный шлюз** — передача неанонимизированного текста в DeepSeek API исключена на уровне кода (Privacy by Design, ФЗ-152).

---

## Структура директорий

```
dockee-backend/
├── cmd/
│   └── server/
│       └── main.go          # точка входа, сборка зависимостей (DI)
├── internal/
│   ├── auth/                # Auth Service: регистрация, JWT, bcrypt
│   ├── document/            # Document Service: загрузка, архив, статусы
│   ├── anonymizer/
│   │   └── ner.go           # NER-анонимизатор: regex для российских ПДн
│   ├── ai/
│   │   └── ai_connector.go  # адаптер DeepSeek API (интерфейс провайдера)
│   ├── pipeline/
│   │   └── pipeline.go      # асинхронный конвейер обработки в горутине
│   ├── repository/          # Repository Layer: PostgreSQL, S3
│   └── audit/               # Audit Middleware: SHA-256 хеши событий
├── migrations/              # SQL-миграции (up/down), golang-migrate
├── docs/                    # OpenAPI 3.0 спецификация (swaggo)
├── .github/
│   └── workflows/
│       └── ci.yml           # CI-пайплайн: lint → test → build
├── docker-compose.yml       # окружение: backend, postgres, minio, migrate
├── .env.example             # шаблон переменных окружения
├── go.mod
├── go.sum
└── CHANGELOG.md             # архитектурные решения по спринтам
```

---

## Локальный запуск

### 1. Клонирование репозитория

```bash
git clone https://github.com/dockee/dockee-backend.git
cd dockee-backend
```

### 2. Переменные окружения

Скопируй шаблон и заполни значения:

```bash
cp .env.example .env
```

Обязательные переменные:

```env
# Сервер
SERVER_PORT=8080

# PostgreSQL
DB_HOST=postgres
DB_PORT=5432
DB_NAME=dockee
DB_USER=dockee_user
DB_PASSWORD=your_password_here

# MinIO (S3)
S3_ENDPOINT=minio:9000
S3_ACCESS_KEY=your_access_key
S3_SECRET_KEY=your_secret_key
S3_BUCKET=dockee-documents
S3_USE_SSL=false

# JWT
JWT_SECRET=your_jwt_secret_min_32_chars
JWT_ACCESS_TTL=15m
JWT_REFRESH_TTL=24h

# DeepSeek API
DEEPSEEK_API_KEY=your_deepseek_api_key
DEEPSEEK_API_URL=https://api.deepseek.com/v1
DEEPSEEK_TIMEOUT_SEC=90

# Среда
APP_ENV=development
```

> ⚠️ Файл `.env` добавлен в `.gitignore` — никогда не коммить реальные секреты.

### 3. Запуск через Docker Compose

```bash
docker-compose up --build
```

Compose поднимает четыре контейнера:

| Контейнер | Описание | Порт |
|---|---|---|
| `backend` | Go-приложение | 8080 |
| `postgres` | PostgreSQL 17 | 5432 |
| `minio` | S3-хранилище | 9000 / 9001 |
| `migrate` | применение миграций при старте | — |

После запуска API доступно по адресу: `http://localhost:8080`

### 4. Проверка работоспособности

```bash
curl http://localhost:8080/health
# {"status":"ok"}
```

Swagger UI с интерактивной документацией:

```
http://localhost:8080/swagger/index.html
```

---

## Разработка

### Запуск тестов

```bash
# юнит-тесты с детектором гонок данных
go test -race ./...

# с отчётом покрытия
go test -race -coverprofile=coverage.out ./...
go tool cover -html=coverage.out
```

Целевое покрытие критических модулей (anonymizer, auth, ai): **≥ 70 %**

### Линтинг

```bash
# Go
golangci-lint run ./...

# проверка безопасности
gosec ./...
```

### Миграции вручную

```bash
# применить все непримененные
migrate -path ./migrations -database "postgres://..." up

# откатить последнюю
migrate -path ./migrations -database "postgres://..." down 1
```

### Сборка бинарника

```bash
go build -o dockee-backend ./cmd/server
```

---

## CI/CD

Пайплайн запускается автоматически при пуше в `dev` и при открытии Pull Request в `main` / `dev`:

```
golangci-lint + gosec  →  go test -race  →  go build
```

Конфигурация: `.github/workflows/ci.yml`

Прямой пуш в `main` запрещён — только через PR с апрувом тимлида и зелёным CI.

---

## Стратегия ветвления

```
main        # стабильный продакшн, только из dev через PR
dev         # интеграционная ветка
feature/*   # feature/{ticket-id}-{описание}
```

Пример имени ветки: `feature/DOC-13-ner-anonymizer`

---

## Связанные репозитории

| Репозиторий | Описание |
|---|---|
| [dockee-frontend](https://github.com/dockee/dockee-frontend) | SPA на Vue.js 3 |
| [dockee-deploy](https://github.com/dockee/dockee-deploy) | docker-compose.yml для продакшна, конфигурации |

API-контракт зафиксирован в `docs/openapi.yaml` и является единственной точкой интеграции между frontend и backend.

---

## Нормативная база

Система обрабатывает юридические документы и персональные данные. Ключевые требования:

- **ФЗ-152** «О персональных данных» — NER-анонимизация как обязательный шлюз
- **ФЗ-149** «Об информации» — хранение и защита данных
- **OWASP Top 10** — gosec, параметризованные SQL-запросы, JWT TTL

---

*Вопросы и предложения — через Issues или Pull Request.*
