<a id="uc-projects.fork"></a>
# UC-projects.fork: Fork demo project into user workspace

| Атрибут | Значение |
|---------|----------|
| **ID** | UC-projects.fork |
| **Уровень** | P0 |
| **Актор** | Аутентифицированный пользователь (любая роль, включая Guest) |
| **Источник** | US-projects.fork |

## Описание

Пользователь создаёт копию (fork) существующего Project (с его парной Ontology) в своём пространстве, получая ссылку на исходный проект (upstream). Fork — базовый механизм для работы с демо-проектами (`VEDO Demos`) и публичными онтологиями.

## Предусловия

1. Пользователь аутентифицирован.
2. Существует source Project с read-доступом для пользователя.
3. Source Project существует в одной из visibility-категорий: Public (любой), Internal (любой аутентифицированный), Private (только members).

## Основной поток

1. Пользователь открывает страницу source Project.
2. Пользователь нажимает «Fork».
3. Система проверяет read-доступ к source Project (membership resolver).
4. Система создаёт новый Project:
   - `upstream_project_id = <source_id>`
   - `visibility = private`
   - В группе пользователя (личное пространство или указанная группа).
5. Система создаёт парную запись в `ontologies` (1:1).
6. Система назначает пользователю роль Owner нового Project.
7. Система вызывает Versioning Service gRPC `CopyBranch(source_ontology_id, "main", new_ontology_id)`.
8. Система возвращает `201 Created` с `{project_id, ontology_id, upstream_project_id}`.
9. Пользователь перенаправляется на страницу нового Project.

## Альтернативные потоки

**3a. Нет read-доступа к source Project:**
- Система возвращает `403 Forbidden`.
- Никаких изменений не производится.

**7a. Versioning Service недоступен:**
- Compensating action: удаление нового Project + Ontology (каскадное удаление).
- Система возвращает `503 Service Unavailable` с сообщением «Service temporarily unavailable, please retry».

**7b. Versioning Service timeout (> 30 сек):**
- Compensating action: удаление нового Project + Ontology.
- Система возвращает `503 Service Unavailable` с сообщением «Copy operation timed out, please retry».

## Постусловия

- Новый Project существует в пространстве пользователя.
- Ontology содержит копию всех entity из source Project на момент fork.
- `upstream_project_id` установлен и ссылается на source Project.
- Аудиторское событие `project.forked` зафиксировано.

## Исключения

- Fork несуществующего source Project → всегда 403 (не 404, per BOLA policy).
- Fork при отсутствии `Idempotency-Key` → 400 Bad Request.
- Fork с повторным `Idempotency-Key` → возвращает существующий fork (идемпотентность).
- Удаление source Project → fork остаётся (orphan fork, `upstream_project_id` сбрасывается в null per `ON DELETE SET NULL`).
