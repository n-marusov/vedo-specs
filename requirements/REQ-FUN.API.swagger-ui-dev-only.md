# Swagger UI — только в dev-окружении

| Атрибут | Значение |
|---------|----------|
| **ID** | REQ-FUN.API.swagger-ui-dev-only |
| **Уровень** | FUN |
| **Атрибут качества** | Interface |
| **Приоритет** | P2 |
| **Статус** | УТВЕРЖДЕНО |
| **Источник** | ADR-DES.API.swagger-ui-dev-only-strategy |
| **Критерии приёмки** | Swagger UI доступен в dev, заблокирован в staging/prod |

---

## Назначение

Предоставить разработчикам и интеграторам интерактивную документацию API через Swagger UI, доступную только в окружении разработчика (dev) через API Gateway, без добавления этой функциональности в staging и production.

## Требование: REQ-FUN.API.swagger-ui-dev-only — Swagger UI только для разработки

### Описание

API Gateway должен обслуживать интерактивный Swagger UI на основе встроенной OpenAPI-спецификации (`/api/v1/openapi.json`) исключительно в dev-окружении. В staging и production окружениях эта функциональность должна быть отключена.

Swagger UI предоставляет:
- Интерактивный просмотр всех REST/GraphQL endpoint-ов
- Возможность отправки тестовых запросов (try-it-out)
- Наглядную схему моделей данных и ответов API
- Удобную навигацию по endpoint-ам с группировкой по тегам

### Детали

| Параметр | Значение |
|----------|----------|
| **URL Swagger UI** | `http://<api-gateway-host>:8080/api/v1/docs` (dev only) |
| **Источник спецификации** | `/api/v1/openapi.json` (встроен через `//go:embed`) |
| **Активация** | Переменная окружения `ENABLE_SWAGGER_UI=true` (по умолчанию `false`) |
| **Dev-окружение** | `ENABLE_SWAGGER_UI=true` в docker-compose.override.yml |
| **Staging/Prod** | Переменная отсутствует или `false`, endpoint возвращает `404` |
| **Способ встраивания** | Swagger UI статика (HTML/JS/CSS) встраивается в бинарник API Gateway через `//go:embed` |
| **Версия Swagger UI** | Фиксируется и вендорится как статика |

### Ожидаемое поведение

| Окружение | `ENABLE_SWAGGER_UI` | `GET /api/v1/docs` | `GET /api/v1/openapi.json` |
|-----------|---------------------|---------------------|---------------------------|
| dev | `true` | Swagger UI (HTML) | OpenAPI JSON |
| staging | `false` или отсутствует | `404 Not Found` | OpenAPI JSON |
| production | `false` или отсутствует | `404 Not Found` | OpenAPI JSON |

### Ограничения

1. **Безопасность:** Swagger UI не должен раскрывать внутренние endpoint-ы, доступные только по gRPC.
2. **Производительность:** Статика Swagger UI не должна увеличивать размер Docker-образа API Gateway более чем на 5 MB.
3. **Изоляция:** Отключение Swagger UI в production не должно влиять на доступность `/api/v1/openapi.json` (raw-спецификация остаётся доступной).
4. **Совместимость:** Спецификация `/api/v1/openapi.json` остаётся публичной во всех окружениях для использования внешними инструментами (Postman, openapi-generator).
