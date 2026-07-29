<a id="uc-org.projects.manage-project-lifecycle"></a>
# UC-org.projects.manage-project-lifecycle: Управление жизненным циклом проектов

| Атрибут | Значение |
|---------|----------|
| **ID** | UC-org.projects.manage-project-lifecycle |
| **Уровень** | P0 |
| **Актор** | Аутентифицированный пользователь с ролью Owner или Maintainer в родительской группе |
| **Источник** | US-org.projects.create |

## Описание

Пользователь управляет жизненным циклом проектов: создаёт проекты в группах через страницу создания, просматривает список, настраивает видимость, обновляет атрибуты, перемещает между группами, удаляет и форкает проекты. Проекты служат контейнерами для онтологий (1:1 Project ↔ Ontology) и обеспечивают разграничение прав доступа, наследование видимости от группы и изоляцию данных.

**Scope текущей реализации:** Создание Project в Group через отдельную страницу. Остальные операции lifecycle (view, update, move, delete, fork) описаны как связанные, но реализуются отдельно.

## Предусловия

1. Пользователь аутентифицирован в системе.
2. Пользователь имеет роль Owner или Maintainer в родительской группе.
3. Родительская группа существует (для создания проекта внутри группы).
4. Система готова к записи (PostgreSQL доступно, auth-service работает).

## Основной поток (создание проекта в группе)

1. Пользователь открывает страницу Projects (`/dashboard/projects`).
2. Система отображает список проектов с человеко-читаемыми именами, видимостью, описанием.
3. Пользователь нажимает «New project».
4. Система перенаправляет на страницу создания проекта `/dashboard/projects/new`.
5. Система отображает страницу «New project» с полями:
   - **Group** — выпадающий список с человеко-читаемыми названиями доступных групп
   - **Project name** — текстовое поле ввода
   - **Project URL/slug** — превью URL (авто-генерация из name, только чтение)
   - **Description** — текстовая область (опционально)
   - **Visibility** — выбор radio-карточками: Private / Internal / Public, по умолчанию Private
6. Пользователь выбирает группу, вводит name, description, выбирает visibility.
7. Пользователь нажимает «Create project».
8. Система валидирует ввод:
   - `name` — обязателен, не пуст (после trim)
   - `group_id` — обязателен, UUID существующей группы
   - `visibility` — одно из: Private, Internal, Public
9. Система отправляет `POST /api/v1/projects` с телом: `{ name, description?, group_id, visibility? }`.
10. API Gateway принимает запрос, маппит `name` (с fallback на `label` для обратной совместимости), передаёт gRPC вызов в auth-service.
11. auth-service проверяет права пользователя в родительской группе (требуется Maintainer или Owner).
12. auth-service валидирует видимость проекта относительно родительской группы (проект не может быть более публичным, чем группа).
13. auth-service создаёт ScopeNode типа `project`:
    - `id` = UUID v4
    - `type` = `project`
    - `parent_id` = `group_id`
    - `name` = из запроса
    - `visibility` = из запроса (по умолчанию `Private`) или унаследованная от группы
14. auth-service генерирует UUID v4 для paired Ontology (`ontology_id`) и создаёт запись в таблице `ontologies`.
15. auth-service назначает пользователя Owner созданного проекта (запись OrgMembership).
16. auth-service фиксирует аудиторское событие `project.created`.
17. API Gateway возвращает `201 Created` с объектом проекта: `{ id, name, type, parent_id, visibility, ontology_id }`.
18. Фронтенд отображает Toast-уведомление об успехе: «Project created successfully».
19. Фронтенд перенаправляет в workspace проекта: `/project/:id/workspace`.
20. Пользователь видит workspace созданного проекта.

## Альтернативные потоки

**8a. name пуст (после trim):**
- Система возвращает inline validation error «Project name is required».
- Проект не создаётся.
- Пользователь остаётся на странице создания.

**8b. name дублируется в пределах группы:**
- Бэкенд возвращает ошибку `POLICY_CONFLICT` (HTTP 409) или код, эквивалентный duplicate.
- Пользователю показывается Toast «Project with this name already exists in this group.».

**8c. group_id отсутствует или невалидный:**
- Система возвращает `INVALID_ARGUMENT` (HTTP 400) — «Group is required».
- Проект не создаётся.

**9a. group_id указывает на несуществующую группу:**
- auth-service возвращает `SCOPE_NOT_FOUND` (HTTP 404).
- Пользователю показывается Toast «Selected group not found».
- Проект не создаётся.

**9b. Недостаточно прав (роль ниже Maintainer в родительской группе):**
- auth-service возвращает `FORBIDDEN_INSUFFICIENT_ROLE` (HTTP 403).
- Пользователю показывается Toast «Insufficient role to create project in this group».
- Проект не создаётся.

**9c. Видимость проекта превышает видимость родительской группы:**
- auth-service возвращает `VISIBILITY_VIOLATION` (HTTP 400/422).
- Пользователю показывается Toast «Visibility cannot exceed parent group».
- Проект не создаётся.

**12a. Сбой создания paired Ontology:**
- auth-service выполняет компенсирующую очистку: удаляет созданный ScopeNode проекта.
- Логируется ошибка: `project.create.ontology_pairing_failed`.
- Клиенту возвращается ошибка `INTERNAL` (HTTP 500).
- Пользователю показывается Toast «Failed to create project. Please try again.».

**9d. Отсутствует Idempotency-Key:**
- Если endpoint защищён idempotency middleware, возвращается `400 Bad Request».
- Рекомендуется повтор с корректным заголовком.

**9e. Повторный запрос с тем же Idempotency-Key:**
- Система возвращает существующий проект (идемпотентность).

## Постусловия

- Новый проект (ScopeNode типа `project`) создан и сохранён в БД (PostgreSQL, таблица `scopes`).
- Paired Ontology с UUID v4 (`ontology_id`) создана и сохранена в таблице `ontologies`.
- Пользователь назначен Owner проекта (запись OrgMembership).
- Аудиторское событие `project.created` зафиксировано.
- Проект отображается в `/dashboard/projects` с человеко-читаемым именем, видимостью, `ontology_id`.
- Toast-уведомление об успешном создании показано пользователю.
- Пользователь перенаправлен в workspace проекта `/project/:id/workspace`.

## Исключения

- Создание проекта с именем, содержащим только пробелы → эквивалентно пустому имени (валидация trim).
- Создание проекта при недоступности auth-service → API Gateway возвращает `503 Service Unavailable`.
- Создание проекта без выбора группы → форма не отправляется, inline validation error.
- Перемещение проекта (move) между группами → отдельный use case, не входит в scope.
- Удаление родительской группы → каскадное удаление дочерних проектов (cascade).
- Форк проекта (fork) → отдельный use case `UC-projects.fork`.
