# Спецификация API и интеграций

| Атрибут | Значение |
|---------|----------|
| **ID** | REQ-FUN.API.integration |
| **Уровень** | FUN |
| **Атрибут качества** | Interface |
| **Приоритет** | P0 |
| **Статус** | УТВЕРЖДЕНО |
| **Источник** | — |
| **Критерии приёмки** | — |

---

## Стратегия API

| Клиент | API | Формат | Обоснование |
|--------|-----|--------|-------------|
| **Frontend (Vue 3)** | GraphQL | JSON | Гибкость, избежание overfetching, Apollo Client |
| **Внешние системы** | REST | JSON | Простота, кэширование, OpenAPI |
| **Внутренние сервисы** | gRPC | Protobuf | Производительность, streaming |
| **Семантические запросы** | SPARQL | Turtle/JSON | RDF standard, W3C |

Граница ответственности API:
- REST endpoints `/api/v1/...` реализуются только на API Gateway как внешний фасад.
- Внутренние сервисы не публикуют функциональные REST-хендлеры для доменных операций.
- API Gateway вызывает Ontology Service, Versioning Service, Auth Service и Metrics Service через gRPC/protobuf contracts.
- HTTP endpoints внутренних сервисов разрешены только для management surface: `GET /`, `GET /health`, `GET /ready`.
- `GET /` возвращает JSON с унифицированными метаданными сервиса и не выполняет бизнес-операции.

## Требование к покрытию API

Критерий "100% операций, доступных через UI, доступны через API" измеряется через Operation Inventory и API Coverage Gate.

Источник правила: `human/artifacts/requirements/REQ-NFR.API.coverage-gate.md`.

Обязательные элементы:
- Все UI-операции перечислены в `docs/operation-inventory.yaml`.
- Каждая операция имеет `api_endpoint`, параметры API, UI equivalent, integration test и `coverage_status`.
- `scripts/check-api-coverage.py` сравнивает inventory с generated `docs/api/openapi.yaml`.
- Merge Request в `main` блокируется, если coverage ниже 100%.
- GitLab CI публикует `api-coverage-report.html` как artifact.

## Лимиты запросов

| Тип клиента | req/min | Burst/10sec | В день |
|-------------|---------|-------------|-------|
| Анонимный | 10 | 5 | 1,000 |
| Пользователь (free) | 100 | 20 | 10,000 |
| Пользователь (pro) | 500 | 100 | 100,000 |
| Enterprise | 1,000 | 200 | 1,000,000 |
| Service account | 2,000 | 500 | 5,000,000 |
| SPARQL endpoint | 10 | 5 | 1,000 |

**Лимиты для отдельных endpoint:**
- `POST /sparql` — 10 req/min (защита от тяжёлых запросов)
- `POST /import` — 5 req/hour
- `GET /export` — 20 req/hour
- `POST /reindex` — 1 req/day

Эти лимиты относятся к публичным endpoint API Gateway. Внутренние gRPC вызовы имеют отдельные server-side limits, deadlines и message-size ограничения.

## Форматы данных

| Сценарий | Формат | MIME type |
|----------|--------|-----------|
| API-запросы/ответы | JSON | `application/json` |
| Ontology export/import | Turtle | `text/turtle` |
| Загрузка файлов | Multipart | `multipart/form-data` |
| Webhooks | JSON | `application/json` |
| SPARQL | SPARQL text | `application/sparql-query` |

## Резервный режим Keycloak

```
Keycloak down → Local DB (emergency users) → Read-only tokens → Emergency mode
```

```yaml
auth:
  primary: keycloak
  fallback:
    enabled: true
    strategies:
      - type: local_db
        priority: 1
        ttl: 3600  # 1 hour tokens
      - type: readonly_tokens
        priority: 2
      - type: emergency_bypass
        priority: 3
        enabled: false
        require_mfa: true
```

## Метрики API

| Метрика | Цель |
|---------|--------|
| p99 REST latency | 200 ms |
| p99 GraphQL latency | 400 ms |
| Availability | 99.9% для MVP SaaS; 99.95% для Enterprise production; 99.99% только Premium Enterprise по согласованию |
| Throughput | 10,000 req/sec |

## Дополнительные решения

| Вопрос | Решение |
|--------|---------|
| WebSocket | Socket.IO с polling fallback |
| Документация OpenAPI | Автогенерация через Swagger UI |
| Версионирование | `/v1/` prefix в URL |
| CORS | Настройка через Kong |
| Batch requests | GraphQL native, REST: `/_batch` |
| Сжатие | Brotli для > 1KB |
| Идемпотентность | `Idempotency-Key` header |

## Нормативная политика идемпотентности

Детальные измеримые требования к write-идемпотентности определены в `api-write-idempotency.md`.

Обязательные пороги:
- 100% покрытие критичных write-операций идемпотентностью.
- >= 99% покрытие всех production write endpoints.
- <= 1% временных исключений с обязательным сроком устранения.
