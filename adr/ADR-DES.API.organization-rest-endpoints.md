# ADR-DES.API.organization-rest-endpoints: Канонический REST-контракт для organization model

**Дата:** 2026-07-21  
**Статус:** Принято

## Контекст

VEDO Core использует GitLab-like модель организации (`Group`, `Project`, `Ontology`, members, visibility, ABAC policies). До настоящего ADR REST-контракт для этой модели был рассеян по трём ADR-ам:

- `ADR-DES.API.protocol-stack-strategy.md` — общее разделение протоколов;
- `ADR-DES.API.graphql-sparql-split-strategy.md` — граница GraphQL/REST;
- `ADR-DES.API.rest-graphql-mutation-boundary.md` — миграция GraphQL mutations в REST.

Ни один из них не фиксирует канонический список REST-путей и их семантику. Одновременно с этим в `ADR-DES.SECURITY.gitlab-like-organization-model.md` была исправлена ошибка идентичности `Project = Ontology`: теперь **Project** — платформенная сущность-контейнер (аналог GitLab Project), а **Ontology** — содержимое Project (TBox/ABox, классы, свойства, индивиды, аксиомы) с жёстким соотношением 1:1. Это делает `members`/`visibility`/`policies` атрибутами **Project**, а не Ontology, и требует канонических REST-путей вида `/projects/:id/...` вместо прежних `/ontologies/:id/...`.

Источником образца для endpoint shapes, access levels, pagination и transfer-семантики является `.ai-factory/references/gitlab-projects-groups-api.md` (GitLab Projects & Groups API). VEDO-специфичные отклонения (в первую очередь 1:1 Project ↔ Ontology) должны быть явно задокументированы как расширения поверх GitLab-базовой линии.

## Требование-источник

- [REQ-NFR.SECURITY.organization-access-model.md](../requirements/REQ-NFR.SECURITY.organization-access-model.md)
- [REQ-FUN.DATA.ontology-visibility-levels.md](../requirements/REQ-FUN.DATA.ontology-visibility-levels.md)
- [ADR-DES.SECURITY.gitlab-like-organization-model.md](ADR-DES.SECURITY.gitlab-like-organization-model.md)
- [ADR-DES.API.write-idempotency-strategy.md](ADR-DES.API.write-idempotency-strategy.md)
- `.ai-factory/references/gitlab-projects-groups-api.md` — GitLab Projects & Groups API (authoritative reference for endpoint shapes, access levels, pagination, transfer semantics).

## Решение

Установить единый канонический REST-контракт для organization model, выровненный по GitLab `/groups/:id/...` и `/projects/:id/...`. Все write-операции идут через REST (см. `ADR-DES.API.rest-graphql-mutation-boundary.md`); GraphQL используется только для read-only навигации по графу онтологии.

### Канонические пути

| Метод | Путь | Назначение |
|-------|------|------------|
| `GET` | `/api/v1/groups` | Список групп, доступных текущему пользователю |
| `POST` | `/api/v1/groups` | Создание группы |
| `GET` | `/api/v1/groups/{id}` | Детали группы |
| `PUT` | `/api/v1/groups/{id}` | Обновление группы |
| `DELETE` | `/api/v1/groups/{id}` | Удаление группы |
| `GET` | `/api/v1/groups/{id}/subgroups` | Подгруппы |
| `GET` | `/api/v1/groups/{id}/members` | Члены группы |
| `POST` | `/api/v1/groups/{id}/members` | Добавить члена |
| `GET` | `/api/v1/projects` | Список проектов, доступных текущему пользователю |
| `POST` | `/api/v1/projects` | Создание Project (с paired Ontology, см. ниже) |
| `GET` | `/api/v1/projects/{id}` | Детали Project (включая `ontology_id`) |
| `PUT` | `/api/v1/projects/{id}` | Обновление Project |
| `DELETE` | `/api/v1/projects/{id}` | Удаление Project (каскадно удаляет paired Ontology) |
| `PUT` | `/api/v1/projects/{id}/move` | Перенос Project в другой Group |
| `GET` | `/api/v1/projects/{id}/members` | Члены Project |
| `POST` | `/api/v1/projects/{id}/members` | Добавить члена Project |
| `PUT` | `/api/v1/projects/{id}/members/{userId}` | Изменить роль члена |
| `DELETE` | `/api/v1/projects/{id}/members/{userId}` | Удалить члена |
| `GET` | `/api/v1/projects/{id}/visibility` | Текущая visibility Project |
| `PUT` | `/api/v1/projects/{id}/visibility` | Изменить visibility Project |
| `GET` | `/api/v1/projects/{id}/policies` | ABAC-политики Project |
| `POST` | `/api/v1/projects/{id}/policies` | Создать политику |
| `DELETE` | `/api/v1/projects/{id}/policies/{policyId}` | Удалить политику |

Форма путей и пагинация соответствуют GitLab-базовой линии из `.ai-factory/references/gitlab-projects-groups-api.md` § «Pagination» (cursor-based, `per_page` ≤ 100). Transfer-семантика `PUT /projects/{id}/move` соответствует § «Transfer a project».

### Project ↔ Ontology pairing (1:1) — VEDO extension

VEDO сохраняет жёсткое соотношение 1:1: один Project содержит ровно одну Ontology, и наоборот. Это **VEDO-специфичное расширение** поверх GitLab-модели: в GitLab Project *является* репозиторием, в VEDO Project — это workspace-контейнер, а Ontology — его графовое содержимое. GitLab не имеет аналога этой связи (см. `.ai-factory/references/gitlab-projects-groups-api.md` § «Create a project» для GitLab-базовой линии).

Правила:

- Создание Project (`POST /api/v1/projects`) атомарно создаёт paired Ontology; возврат содержит `ontology_id`.
- Удаление Project (`DELETE /api/v1/projects/{id}`) каскадно удаляет paired Ontology.
- Ontology достигается через `GET /api/v1/ontologies/{id}`, где `{id}` — идентификатор Ontology (не Project id).
- Прямое отображение Project → Ontology экспонируется через `ProjectDetail.ontology_id`; обратное отображение — через `OntologyDetail.project_id`.
- Группировка нескольких онтологий выполняется через иерархию `Group`, а не через упаковку в один Project.

### Scope strings в auth-service gRPC-контракте

Auth-service использует строковые scope-идентификаторы:

- `"group/" + id` — для Group;
- `"project/" + id` — для Project (canonical);
- `"ontology/" + id` — **legacy alias**, принимается только в окне миграции настоящего плана и удаляется в финальной cleanup-задаче (Task 7.4 плана `feature/project-ontology-separation`).

### Idempotency

Все write-эндпоинты принимают заголовок `Idempotency-Key` (REQ-FUN.API.write-idempotency, `ADR-DES.API.write-idempotency-strategy`). Эндпоинты members, visibility и policies **требуют** его наличия — отсутствие ключа возвращает `400 INVALID_IDEMPOTENCY_KEY`. Это предотвращает duplicate-вставки членства и повторные изменения visibility/policies при сетевых ретрях.

### RBAC

- Только **Owner** может управлять members, visibility и policies (см. `ADR-DES.SECURITY.gitlab-like-organization-model.md`).
- **Maintainer** сфокусирован на workflow онтологии (protected branches, Ontology Merge Request, approvals) и **не может** изменять org-endpoints.
- Уровни доступа соответствуют GitLab per `.ai-factory/references/gitlab-projects-groups-api.md` § «Access Levels Reference»: `Guest=10`, `Reporter=20`, `Developer=30`, `Maintainer=40`, `Owner=50`.
- VEDO legacy-алиасы `Viewer`/`Editor` отображаются на `Guest`/`Developer` соответственно (см. Antora `organization-model.adoc` § «Role Hierarchy»).

### Audit

Каждый write-эндпоинт эмittит структурированную строку в `audit_events`:

| Поле | Значение |
|------|----------|
| `event` | `member.added`, `member.removed`, `member.role_changed`, `visibility.changed`, `policy.created`, `policy.deleted`, `project.created`, `project.deleted`, `project.moved`, `group.created`, `group.deleted` |
| `reason` | optional human-readable reason |
| `user_id` | инициатор |
| `object_type` | `project` \| `group` |
| `object_id` | идентификатор Project или Group |
| `source_ip` | IP инициатора |
| `trace_id` | OpenTelemetry trace id |
| `timestamp` | ISO 8601 UTC |

## Рассмотренные альтернативы

**1. `Ontology` как REST-носитель для members/visibility/policies (status quo до настоящего ADR).**  
Отклонено — конflateирует платформенный контейнер с графовым содержимым и нарушает GitLab-выравнивание. В GitLab members живут на `/projects/:id/members`, а не на пути репозиториального содержимого (см. `.ai-factory/references/gitlab-projects-groups-api.md` § «Project members»). Кроме того, это противоречит 1:1 Project ↔ Ontology: атрибут доступа должен храниться на Project, а Ontology — наследовать через 1:1-связь.

**2. Алиасы `/ontologies/{id}/members` ↔ `/projects/{id}/members`.**  
Отклонено — при 1:1 они лишь маскируют модель и усложняют routing audit/idempotency (какой из двух путей считать каноническим для дедупликации `Idempotency-Key`?). Один канонический путь — `/projects/{id}/members`.

**3. Вложенность members под `/groups/{id}/projects/{id}/members`.**  
Отклонено как over-nesting. GitLab держит `/projects/:id/members` плоско (см. `.ai-factory/references/gitlab-projects-groups-api.md` § «Project members»), и VEDO следует этому правилу.

**4. Отклонение от GitLab access level integers.**  
Отклонено — чтобы сохранить интуитивное role-mapping для пользователей, приходящих из GitLab. Числовая лестница (10/20/30/40/50) сохранена как внутренняя authorization-шкала, даже though REST-payloads используют строковые role-имена.

## Последствия

**Положительные:**

- Единый канонический REST-контракт для organization model — endpoint shapes больше не рассредоточены по трём ADR-ам.
- GitLab-выравнивание упрощает онбординг пользователей и интеграцию с инструментами, ожидающими GitLab-like API.
- 1:1 Project ↔ Ontology явно задокументировано как VEDO extension — у разработчиков и AI-агентов есть однозначный контракт.
- Чёткое разделение: members/visibility/policies на Project; workflow онтологии на Maintainer; audit/idempotency на всех write-операциях.

**Отрицательные:**

- Existing integrations, использующие `/ontologies/{id}/members|visibility|policies`, требуют миграции на `/projects/{id}/...`. Окно миграции покрывается legacy `"ontology/" + id` scope alias в auth-service.
- `POST /ontologies` помечается как deprecated (но не удаляется в этом плане) — требуется follow-up план после deprecation window.

**Меры снижения рисков:**

- Migration plan в `ADR-DES.API.rest-graphql-mutation-boundary.md` обновлён синхронно (Task 1.4 плана `feature/project-ontology-separation`).
- Frontend (Phase 6 плана) переключается на `/projects/{id}/...` с transparent resolution `projectId` из `ontologyId` через `OntologyDetail.project_id`.
- Security negative tests (Task 7.1 плана) доказывают, что bypass закрыт для renamed endpoints (real HTTP requests, no mocks).

## Связанные ADR

- [ADR-DES.SECURITY.gitlab-like-organization-model.md](ADR-DES.SECURITY.gitlab-like-organization-model.md) — модель организации (Group/Project/Ontology 1:1, роли, наследование).
- [ADR-DES.API.protocol-stack-strategy.md](ADR-DES.API.protocol-stack-strategy.md) — общее разделение протоколов; строка «Управление орг. моделью» ссылается сюда.
- [ADR-DES.API.graphql-sparql-split-strategy.md](ADR-DES.API.graphql-sparql-split-strategy.md) — граница GraphQL/REST; таблица ответственности обновлена взаимно.
- [ADR-DES.API.rest-graphql-mutation-boundary.md](ADR-DES.API.rest-graphql-mutation-boundary.md) — миграция GraphQL mutations; таблица миграции обновлена взаимно.
- [ADR-DES.API.write-idempotency-strategy.md](ADR-DES.API.write-idempotency-strategy.md) — `Idempotency-Key` для write-эндпоинтов.

---
