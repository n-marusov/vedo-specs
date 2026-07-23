# ADR-DES.DATA.uuid-identifiers-for-groups-projects-mandate: UUID идентификаторы для групп и проектов

**Дата:** 2026-07-23
**Статус:** Принято

## Контекст

В проекте VEDO Core онтологии идентифицируются через UUID (`ontology_id`). Это обеспечивает глобальную уникальность, отсутствие коллизий при слиянии форков, и соответствует семантике распределённой версионности (Git-like).

В то же время, для групп (groups) и проектов (projects) в модели организации (organization model) изначально рассматривались альтернативы:
- Числовые автоинкрементные ID (BIGSERIAL)
- Slug-идентификаторы (human-readable, e.g., `my-org/my-project`)
- UUID v4

Числовые ID создают риск коллизий при федерации/миграции данных между инстансами и несовместимы с Git-like семантикой форков. Slug-идентификаторы требуют координации уникальности, нестабильны при переименовании и усложняют разрешение ссылок в распределённой среде.

## Требование-источник

- `specs/adr/ADR-DES.API.organization-rest-endpoints.md` — канонические пути API используют `group_id`, `project_id`
- `specs/adr/ADR-DES.INFRA.ontology-publishing.md` — онтологии используют UUID

## Решение

**Идентификаторы групп (`group_id`) и проектов (`project_id`) обязаны быть UUID v4 (RFC 4122).**

- Формат: стандартный UUID v4 (36 символов, 5 групп через дефис: `xxxxxxxx-xxxx-4xxx-yxxx-xxxxxxxxxxxx`)
- Генерация: на стороне сервиса (auth-service / organization-service) при создании сущности
- Клиенты **не могут** задавать свой ID — сервер всегда генерирует
- В API пути: `/groups/{group_id}`, `/projects/{project_id}` — `group_id` и `project_id` валидируются как UUID
- В gRPC/Protobuf контрактах: `string group_id = 1 [(validate.rules).string.uuid = true]`
- В БД (PostgreSQL): колонки `uuid` типа `uuid` с `DEFAULT gen_random_uuid()`
- В Neo4j (если узлы групп/проектов хранятся): свойство `uuid` с уникальным ограничением

Онтологии (`ontology_id`) продолжают использовать UUID v4 — единый стандарт идентификации для всех сущностей первого класса.

## Рассмотренные альтернативы

| Вариант | Плюсы | Минусы | Вердикт |
|---------|-------|--------|---------|
| **UUID v4 (принято)** | Глобальная уникальность без координации, стабилен при переименовании, нативно поддерживается PG/Neo4j, совместим с Git-like fork/merge | Нечитаем человеком, 36 байт в URL | ✅ Принят |
| BIGSERIAL / BIGINT | Компактен, быстрый JOIN | Коллизии при федерации/миграции, требует центрального последователя, нестабилен при форках | ❌ Отклонён |
| Slug (org/project) | Человекочитаем, SEO-friendly | Требует уникальности в скоупе, ломается при rename, сложно в распределённых системах | ❌ Отклонён (только как display-name/alias) |
| UUID v7 (timestamp-based) | Сортируемо по времени, уникальность | Ещё не стандартизирован широко, сложнее генерация | ⏳ Рассмотрено, отложено |

## Последствия

### Положительные
- **Единый стандарт идентификации** — онтологии, группы, проекты, коммиты (versioning-service) все используют UUID
- **Федерация-ready** — никакой координации ID между инстансами не требуется
- **Форк-совместимость** — форк проекта сохраняет историю через project_id, новый форк получает новый UUID
- **Безопасность** — непредсказуемые ID затрудняют перебор (enumeration attack)

### Отрицательные
- URL становятся длиннее и менее читаемы для человека
- Нужны UI-алиасы (slug/display name) для удобства пользователей

### Меры снижения рисков
- Во фронтенде использовать `display_name` / `slug` для отображения и навигации, UUID — только для внутренних ссылок и API
- Добавить валидацию UUID в OpenAPI/GraphQL схемах и клиентских SDK
- Документировать в API reference, что `group_id` и `project_id` — это UUID

## Связанные ADR

- [ADR-DES.API.organization-rest-endpoints.md](ADR-DES.API.organization-rest-endpoints.md) — канонический REST-контракт для organization model (`/groups/:id/...`, `/projects/:id/...`)
- [ADR-DES.INFRA.ontology-publishing.md](ADR-DES.INFRA.ontology-publishing.md) — онтологии используют UUID
- [ADR-DES.SECURITY.gitlab-like-organization-model.md](ADR-DES.SECURITY.gitlab-like-organization-model.md) — GitLab-like модель организации

---