# Group CRUD

| Атрибут | Значение |
|---------|----------|
| **ID** | REQ-FUN.ORG.group-crud |
| **Уровень** | FUN |
| **Атрибут качества** | Organization |
| **Приоритет** | P0 |
| **Статус** | УТВЕРЖДЕНО |
| **Источник** | ADR-DES.SECURITY.gitlab-like-organization-model |
| **Критерии приёмки** | См. раздел «Критерии приёмки» |

---

## Описание

Система MUST поддерживать CRUD-операции для групп с иерархической структурой (группы, подгруппы до 5 уровней вложенности) и управлением видимостью (visibility). Группы служат контейнерами для проектов и обеспечивают наследование прав доступа для участников.

## Функциональные блоки

### 1. Создание группы (Create Group)

| Атрибут | Описание |
|---------|----------|
| **Поле name** | Обязательное, строка. Не может быть пустой или состоять только из пробелов. Каноническое имя поля в REST API — `name` (поле `label` — устаревшее, поддерживается для обратной совместимости). |
| **Поле slug** | Авто-генерируется бэкендом из name (lowercase, транслитерация латиницей, замена пробелов на `-`). Уникален в рамках родительской группы. |
| **Поле visibility** | Optional. Одно из: `private` (по умолчанию), `internal`, `public`. Для подгруппы: наследуется от родителя, если не указана явно. Подгруппа не может быть более публичной, чем родитель. |
| **Поле parent_id** | Optional. UUID родительской группы. Если указан — создаётся подгруппа. |
| **Поле description** | Optional. Текстовое описание группы. |
| **Роль для создания** | Owner. |

**HTTP:** `POST /api/v1/groups` → `201 Created` с объектом группы.

**Запрос:** `{ name, slug?, description?, visibility?, parent_id? }`.

**Ответ:** `{ id, name, slug, description, visibility, parent_id, path, created_at }`.

**После успешного создания:** фронтенд перенаправляет на `/dashboard/groups/:id` и показывает Toast-уведомление.

**Устаревшие поля:** `label` (альтернативное имя для name, только обратная совместимость), `invite_members` (запланировано на будущую итерацию).

**Валидация:**
- name — required, trim, не пуст (каноническое поле `name`, устаревшее `label` — fallback)
- slug — генерируется бэкендом из name, проверка уникальности в рамках родителя
- visibility — одно из: Private, Internal, Public (enum)
- parent_id — если указан, должен существовать и быть группой
- Глубина иерархии — не более 5 уровней
- Циклические parent_id — запрещены
- Видимость подгруппы — не может превышать видимость родителя (Private > Internal > Public)
- Если visibility не указана для подгруппы — наследуется от родителя

### 2. Чтение списка групп (List Groups)

| Атрибут | Описание |
|---------|----------|
| **Фильтрация** | Поиск по name через query-параметр `?search=`. |
| **Иерархия** | Подгруппы возвращаются как вложенные children родительской группы. |
| **Пагинация** | Поддерживается через `pagination` параметры. |
| **Поля ответа** | `name`, `slug`, `parent_id`, `visibility`, `child_count`, `member_count`, `project_count`. |

**HTTP:** `GET /api/v1/groups` → `200 OK` с массивом групп.

### 3. Получение группы (Get Group)

**HTTP:** `GET /api/v1/groups/:id` → `200 OK` с объектом группы.

### 4. Обновление группы (Update Group)

| Атрибут | Описание |
|---------|----------|
| **Поле name** | Обновляемое. |
| **Поле description** | Обновляемое. |
| **Поле visibility** | Обновляемое. |
| **Роль для обновления** | Maintainer или выше. |

**HTTP:** `PUT /api/v1/groups/:id` → `200 OK` с обновлённым объектом.

### 5. Удаление группы (Delete Group)

| Атрибут | Описание |
|---------|----------|
| **Каскадное удаление** | Подгруппы и проекты удаляются каскадно. |
| **Роль для удаления** | Owner. |
| **Защита** | Нельзя удалить группу, если пользователь — единственный Owner в скоупе (Last Owner Removal Blocked). |

**HTTP:** `DELETE /api/v1/groups/:id` → `204 No Content`.

### 6. Список подгрупп (List Child Groups)

**HTTP:** `GET /api/v1/groups/:id/subgroups` → `200 OK` с массивом дочерних групп.

## Visibility Semantics

| Значение | Кто видит группу и проекты |
|----------|---------------------------|
| **Private** | Только участники группы (members). |
| **Internal** | Любой аутентифицированный пользователь, кроме external-пользователей. |
| **Public** | Любой пользователь без аутентификации. |

## Критерии приёмки

- [ ] `POST /api/v1/groups` с валидными данными возвращает `201 Created` с UUID группы.
- [ ] `POST /api/v1/groups` с полем `name` (каноническое) создаёт группу.
- [ ] `POST /api/v1/groups` с полем `label` (устаревшее, fallback) создаёт группу с WARN-логированием.
- [ ] `POST /api/v1/groups` без name возвращает `400 Bad Request` с ошибкой валидации.
- [ ] После успешного создания: фронтенд перенаправляет на `/dashboard/groups/:id` и показывает Toast.
- [ ] `GET /api/v1/groups` возвращает иерархический список групп с полями `name`, `slug`, `visibility`, `child_count`.
- [ ] `GET /api/v1/groups?search=Engineering` фильтрует группы по имени.
- [ ] `GET /api/v1/groups/:id` возвращает детальную информацию о группе (`name`, `slug`, `path`, `visibility`, `children`).
- [ ] `PUT /api/v1/groups/:id` обновляет name и visibility.
- [ ] `DELETE /api/v1/groups/:id` удаляет группу и каскадно подгруппы.
- [ ] Создание подгруппы с parent_id существующей группы работает.
- [ ] Подгруппа не может быть более публичной, чем родитель (Private > Internal > Public).
- [ ] Если visibility не указана для подгруппы — наследуется от родителя.
- [ ] Создание группы с несуществующим parent_id возвращает `SCOPE_NOT_FOUND`.
- [ ] Создание группы с глубиной >5 возвращает `HIERARCHY_DEPTH_EXCEEDED`.
- [ ] Создание группы с циклическим parent_id возвращает `CYCLE_DETECTED`.
- [ ] Пользователь без роли Owner не может создать группу (`FORBIDDEN_INSUFFICIENT_ROLE`).
- [ ] Realm-роли Keycloak обрабатываются case-insensitive: `"owner"`, `"Owner"`, `"OWNER"` — все разрешаются в weight 3.
- [ ] Visibility enum валидируется: допускаются только Private, Internal, Public.
- [ ] Идемпотентность: повторный `POST /groups` с тем же `Idempotency-Key` возвращает существующую группу.

## Связанные артефакты

- **User Story:** `US-org.groups.create`
- **Use Case:** `UC-org.groups.manage-group-lifecycle`
- **ADR-источник:** `ADR-DES.SECURITY.gitlab-like-organization-model`
- **ADR-ID:** `ADR-DES.DATA.uuid-identifiers-for-groups-projects-mandate`
- **GUI Design:** `specs/ui/gui-tree.yaml` — GroupsPage + Create Group dialog
- **Frontend:** `GroupsPage.vue`, `CreateGroupDialog.vue`, `org.ts`
- **API Gateway:** `routes.go`, `handlers/org_handler.go`, `proxy/grpc_org_client.go`, `auth/auth.go`
- **Auth-service:** `org/org.go`, `org/types.go`, `org/store.go`, `org/postgres_store.go`

## История изменений

| Версия | Дата | Автор | Изменения |
|--------|------|-------|-----------|
| v1.0 | 2026-05-16 | Security Architect | Initial specification |
| v1.1 | 2026-07-25 | Agent | Исправлена case-sensitive проверка ролей в API Gateway. Realm-роли Keycloak
  (`"owner"`, `"editor"`, `"viewer"`, etc.) — нижний регистр. Карта `roleWeight`
  в `auth.go` теперь использует lowercase-ключи. Добавлены недостающие роли:
  `"reviewer"` (weight 1), `"admin"` (weight 3), `"service"` (weight 3).
  Регрессионные тесты: `TestKeycloak_LowercaseRealmRole_OwnerCanPost`,
  `TestKeycloak_LowercaseRealmRole_ViewerBlocked`.
