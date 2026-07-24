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
| **Поле name** | Обязательное, строка. Не может быть пустой или состоять только из пробелов. |
| **Поле slug** | Авто-генерируется из name (lowercase, транслитерация латиницей, замена пробелов на `-`). Уникален в рамках родительской группы. |
| **Поле visibility** | Optional. Одно из: `private` (по умолчанию), `internal`, `public`. |
| **Поле parent_id** | Optional. UUID родительской группы. Если указан — создаётся подгруппа. |
| **Поле usage** | Optional. `team` или `solo`. |
| **Поле invite_members** | Optional. Email-адреса приглашаемых участников. |
| **Роль для создания** | Owner. |

**HTTP:** `POST /api/v1/groups` → `201 Created` с объектом группы.

**Валидация:**
- name — required, trim, не пуст
- slug — генерируется из name, проверка уникальности
- visibility — одно из: Private, Internal, Public (enum)
- parent_id — если указан, должен существовать и быть группой
- Глубина иерархии — не более 5 уровней
- Циклические parent_id — запрещены

### 2. Чтение списка групп (List Groups)

| Атрибут | Описание |
|---------|----------|
| **Фильтрация** | Поиск по name через query-параметр `?search=`. |
| **Иерархия** | Подгруппы возвращаются как вложенные children родительской группы. |
| **Пагинация** | Поддерживается через `pagination` параметры. |

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
- [ ] `POST /api/v1/groups` без name возвращает `400 Bad Request` с ошибкой валидации.
- [ ] `GET /api/v1/groups` возвращает иерархический список групп.
- [ ] `GET /api/v1/groups?search=Engineering` фильтрует группы по имени.
- [ ] `PUT /api/v1/groups/:id` обновляет name и visibility.
- [ ] `DELETE /api/v1/groups/:id` удаляет группу и каскадно подгруппы.
- [ ] Создание подгруппы с parent_id существующей группы работает.
- [ ] Создание группы с несуществующим parent_id возвращает `SCOPE_NOT_FOUND`.
- [ ] Создание группы с глубиной >5 возвращает `HIERARCHY_DEPTH_EXCEEDED`.
- [ ] Создание группы с циклическим parent_id возвращает `CYCLE_DETECTED`.
- [ ] Пользователь без роли Owner не может создать группу (`FORBIDDEN_INSUFFICIENT_ROLE`).
- [ ] Visibility enum валидируется: допускаются только Private, Internal, Public.
- [ ] Идемпотентность: повторный `POST /groups` с тем же `Idempotency-Key` возвращает существующую группу.

## Связанные артефакты

- **User Story:** `US-org.groups.create`
- **Use Case:** `UC-org.groups.manage-group-lifecycle`
- **ADR-источник:** `ADR-DES.SECURITY.gitlab-like-organization-model`
- **ADR-ID:** `ADR-DES.DATA.uuid-identifiers-for-groups-projects-mandate`
- **GUI Design:** `specs/ui/gui-tree.yaml` — GroupsPage + Create Group dialog
- **Frontend:** `GroupsPage.vue`, `CreateGroupDialog.vue`, `org.ts`
- **API Gateway:** `routes.go`, `handlers/org_handler.go`, `proxy/grpc_org_client.go`
- **Auth-service:** `org/org.go`, `org/types.go`, `org/store.go`, `org/postgres_store.go`
