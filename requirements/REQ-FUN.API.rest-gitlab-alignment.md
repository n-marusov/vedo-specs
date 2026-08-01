# Выравнивание REST API по GitLab

| Атрибут | Значение |
|---------|----------|
| **ID** | REQ-FUN.API.rest-gitlab-alignment |
| **Уровень** | FUN |
| **Атрибут качества** | Interface |
| **Приоритет** | P0 |
| **Статус** | ЧЕРНОВИК |
| **Источник** | Исследовательская сессия 2026-08-01: REST API realignment to GitLab; ADR-DES.API.organization-rest-endpoints |

---

## Назначение

Определить целевую структуру REST API, выровненную по конвенциям GitLab: все операции чтения и записи вложены под `/projects/{pid}/`. Плоские пути `/api/v1/ontologies/*` признаны устаревшими и мигрируются в целевую структуру.

## Требование

REST API использует GitLab-выровненную структуру путей:

**Канонические пути:**

| Метод | Путь | Назначение |
|-------|------|------------|
| `GET` / `POST` | `/api/v1/projects` | Список / создание проектов |
| `GET` / `PUT` / `DELETE` | `/api/v1/projects/{pid}` | CRUD проекта |
| `GET` / `POST` | `/api/v1/projects/{pid}/repository/classes` | CRUD классов с `?ref=` |
| `GET` / `POST` | `/api/v1/projects/{pid}/repository/properties` | CRUD свойств с `?ref=` |
| `GET` / `POST` | `/api/v1/projects/{pid}/repository/individuals` | CRUD индивидов с `?ref=` |
| `GET` | `/api/v1/projects/{pid}/repository/commits` | История коммитов |
| `GET` | `/api/v1/projects/{pid}/repository/commits/{sha}/diff` | Семантический diff |
| `GET` / `POST` | `/api/v1/projects/{pid}/repository/branches` | Управление ветками |
| `GET` / `POST` | `/api/v1/projects/{pid}/merge_requests` | CRUD MR (iid-based) |
| `PUT` | `/api/v1/projects/{pid}/merge_requests/{merge_request_iid}/merge` | Merge MR |
| `GET` | `/api/v1/projects/{pid}/releases` | Опубликованные снимки (releases) |
| `GET` / `POST` | `/api/v1/projects/{pid}/protected_branches` | Защита веток |
| `POST` | `/api/v1/projects/{pid}/fork` | Fork проекта (уже существует) |

**Глобальные эндпоинты:**

- Разрешены только read-агрегации: список проектов, поиск по проектам.
- Никаких write-операций на глобальном уровне.

**Параметр `?ref=`:**

- Контентные пути используют `?ref=` (GitLab-паттерн `ref_name`) для выбора ветки.
- REST write-эндпоинты работают с branch-local снимками (`branch snapshots`), а не с живой онтологией.

## Миграция устаревших путей (Deprecation)

- Пути `/api/v1/ontologies/*` помечаются заголовками `Deprecation: true` и `Sunset: ...` (см. REQ-FUN.API.sunset-header).
- Старые пути после deprecation window возвращают `501 Not Implemented` с заголовком `x-vedo-status: planned`.
- `ontology_id` остаётся внутренним идентификатором и НЕ экспонируется в REST-поверхности как канонический путь.
- GraphQL: аргумент `ontology_id` → `project_id` (+ опционально `branch`).

## Обратная совместимость

- Старые пути возвращают `501 Not Implemented` с заголовком `x-vedo-status: planned` (контракт для потребителей API).
- Новые пути не дублируют старые — используется единая каноническая структура (без dual paths).

## Критерии приёмки

1. Все write/read операции вложены под `/projects/{pid}/`.
2. `/api/v1/ontologies/*` помечены как deprecated (заголовки `Deprecation`/`Sunset`).
3. Старые пути возвращают `501` с `x-vedo-status: planned`.
4. Глобальные эндпоинты — только read-агрегации.
5. Все новые пути включены в BOLA/BFLA authorization tables.
6. OpenAPI spec (`api-gateway/docs/openapi.json`) отражает целевую структуру.

## Ссылки (References)

- [ADR-DES.API.rest-gitlab-alignment.md](../adr/ADR-DES.API.rest-gitlab-alignment.md) — архитектурное решение о GitLab-выравнивании
- [ADR-DES.API.organization-rest-endpoints.md](../adr/ADR-DES.API.organization-rest-endpoints.md) — канонический REST-контракт organization model
- [ADR-DES.API.rest-graphql-mutation-boundary.md](../adr/ADR-DES.API.rest-graphql-mutation-boundary.md) — граница GraphQL/REST
- GitLab API documentation — эталон для endpoint shapes (`.ai-factory/references/gitlab-projects-groups-api.md`)

---

## Обоснование (Rationale)

Текущие пути `/api/v1/ontologies/{id}/*` — наследие до project separation. GitLab-модель диктует вложенные пути под `/projects/{pid}/`, а плоские ontology-пути противоречат модели 1:1 Project ↔ Ontology и запутывают API-поверхность. Выравнивание по GitLab делает API предсказуемым и консистентным с organization model. Миграция отложена до REST API GitLab Alignment Migration (M10 — versioning, M11 — CRUD/publishing); в текущем плане применяются только guardrails (deprecation headers, planned stubs).
