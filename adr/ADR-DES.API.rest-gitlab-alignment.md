# ADR-DES.API.rest-gitlab-alignment

**Дата:** 2026-08-01  
**Статус:** Принято

## Контекст

Текущие пути `/api/v1/ontologies/{id}/*` — наследие до project separation. GitLab-модель диктует вложенные пути под `/projects/{pid}/`; плоские ontology-пути противоречат GitLab и модели 1:1 Project ↔ Ontology (см. `ADR-DES.SECURITY.gitlab-like-organization-model.md`).

Проблемы текущей структуры:
- Плоские `/ontologies/{id}/...` пути конфликтуют с GitLab-конвенциями и организационной моделью.
- Нет единого контракта для repository content, MR, releases, protected branches.
- Глобальные эндпоинты смешивают read-агрегации с write-операциями.

Источник образца: `.ai-factory/references/gitlab-projects-groups-api.md` (GitLab Projects & Groups API) — проверено против официальной документации GitLab.

## Требование-источник

- [REQ-FUN.API.rest-gitlab-alignment.md](../requirements/REQ-FUN.API.rest-gitlab-alignment.md)
- [ADR-DES.API.organization-rest-endpoints.md](ADR-DES.API.organization-rest-endpoints.md) — канонический REST-контракт organization model
- [ADR-DES.API.rest-graphql-mutation-boundary.md](ADR-DES.API.rest-graphql-mutation-boundary.md) — граница GraphQL/REST

## Решение

**Принять GitLab-совместимую структуру REST API: все операции вложены под `/projects/{pid}/`.**

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

**Правила:**

- Контентные пути (`repository/classes|properties|individuals`) используют `?ref=` (GitLab `ref_name` паттерн).
- MR-жизненный цикл — 1:1 с GitLab: iid-based, `PUT .../merge`, `state_event`, `/diffs`.
- Глобальные эндпоинты — только read-агрегации (список проектов, поиск).
- `ontology_id` остаётся внутренним идентификатором (не экспонируется в REST-поверхности).
- GraphQL: аргумент `ontology_id` → `project_id` (+ опционально `branch`).

### План миграции

1. Старые пути (`/api/v1/ontologies/*`) → помечаются `Deprecation`/`Sunset` заголовками.
2. После deprecation window → `501 Not Implemented` с заголовком `x-vedo-status: planned`.
3. OpenAPI spec (`api-gateway/docs/openapi.json`) — полный rewrite под целевую структуру.
4. Frontend API clients и 7 сервисов переключаются на `/projects/{pid}/...`.

### Endpoint-class audit (BOLA/BFLA)

Все новые пути обязаны быть включены в BOLA/BFLA authorization tables:

| Endpoint class | Пример | Требуемая роль | Примечание |
|----------------|--------|----------------|------------|
| Project reads | `GET /projects/{pid}` | Guest+ (по visibility) | BOLA-sensitive |
| Project writes | `PUT /projects/{pid}` | Maintainer+ | — |
| Repository content reads | `GET /projects/{pid}/repository/classes?ref=` | Guest+ (branch visibility) | BOLA-sensitive |
| Repository content writes | `POST /projects/{pid}/repository/classes?ref=` | Developer+ (branch-local) | BFLA-sensitive |
| MR lifecycle | `PUT /projects/{pid}/merge_requests/{iid}/merge` | Maintainer+ | BFLA-sensitive |
| Releases | `GET /projects/{pid}/releases` | public/restricted per snapshot | — |
| Protected branches | `POST /projects/{pid}/protected_branches` | Maintainer+ | BFLA-sensitive |

## Рассмотренные альтернативы

| Альтернатива | Причина отклонения |
|--------------|--------------------|
| A) Сохранить текущие плоские пути | Противоречит GitLab-модели, запутанная API-поверхность |
| B) Dual paths (старые + новые) | Непоследовательно, удваивает тестовую поверхность |
| C) Полное GitLab-выравнивание (выбрано) | Консистентно, предсказуемо, соответствует organization model |

## Последствия

**Положительные:**
- Единый предсказуемый контракт, выровненный по отраслевому стандарту (GitLab).
- Упрощает онбординг пользователей и интеграцию с GitLab-совместимыми инструментами.
- Чёткая декомпозиция: repository content / MR / releases / protected branches.

**Отрицательные:**
- openapi.json — полный rewrite.
- Затронуты 7 сервисов и 84+ тестовых файлов.
- Antora docs и traceability.ttl требуют обновления.
- Frontend API clients требуют миграции.

**Меры снижения рисков:**
- Guardrails (deprecation headers, planned stubs) применяются до миграции (REST API GitLab Alignment Migration, M10/M11).
- Endpoint-class таблица обеспечивает BOLA/BFLA-покрытие новых путей.
- Поэтапная миграция сервисов с сохранением обратной совместимости через `x-vedo-status: planned`.

## Связанные ADR

- [ADR-DES.API.organization-rest-endpoints.md](ADR-DES.API.organization-rest-endpoints.md) — канонический REST-контракт organization model
- [ADR-DES.API.rest-graphql-mutation-boundary.md](ADR-DES.API.rest-graphql-mutation-boundary.md) — граница GraphQL/REST
- [ADR-DES.API.write-path-invariant.md](ADR-DES.API.write-path-invariant.md) — branch-local snapshots для write-путей
- [ADR-DES.SECURITY.gitlab-like-organization-model.md](ADR-DES.SECURITY.gitlab-like-organization-model.md) — 1:1 Project ↔ Ontology

---
