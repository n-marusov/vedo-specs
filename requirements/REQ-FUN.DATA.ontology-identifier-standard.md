# Стандарт идентификаторов онтологий (UUID)

| Атрибут | Значение |
|---------|----------|
| **ID** | REQ-FUN.DATA.ontology-identifier-standard |
| **Уровень** | FUN |
| **Атрибут качества** | Maintainability, Interoperability |
| **Приоритет** | P1 |
| **Статус** | ЧЕРНОВИК |
| **Источник** | Code Review: `ontology_id` не UUID (Project ↔ Ontology Separation) |
| **Связанные ADR** | `ADR-DES.API.organization-rest-endpoints`, `ADR-DES.SECURITY.gitlab-like-organization-model` |

---

## Назначение

Определить стандарт идентификаторов для сущностей Ontology в VEDO Core. Ontology должна использовать UUID v4 в качестве первичного идентификатора (`ontology_id`) вместо адресных/составных строк (например, scope-based id вида `"project/" + name`).

## Требование

При создании Project (и его paired Ontology) идентификатор онтологии `ontology_id` должен быть UUID v4 по RFC 4122, сгенерированный криптостойким генератором случайных чисел (например, `crypto/rand` в Go).

### Правила

1. **UUID v4 — единственный формат** `ontology_id`. Адресные идентификаторы (scope id, человекочитаемые строки) не допускаются.
2. **Генерация на стороне сервера** — `ontology_id` создаётся в auth-service при создании Project и сохраняется в таблице `ontologies`.
3. **Криптостойкий генератор** — использование `crypto/rand` (Go) или эквивалента обязательно. `math/rand`, `time.Now().UnixNano()` и прочие не-криптостойкие источники запрещены.
4. **Обратная совместимость** — существующие записи в `ontologies`, созданные с `ontology_id = project_scope` (legacy identity из migration 009), сохраняются как есть. UUID-формат применяется только к новым проектам, созданным после внедрения этого стандарта.

### Критерии приёмки

1. `POST /api/v1/projects` возвращает `ontology_id` в формате UUID v4, а не `"project/" + name`.
2. Сгенерированный UUID парсится валидатором UUID без ошибок.
3. `GET /api/v1/ontologies/{id}` принимает UUID в качестве онтологий.
4. Существующие legacy-записи (с `ontology_id = "project/..."`) продолжают работать — они не пересоздаются.
5. `ontology-service` может идентифицировать онтологию по UUID.

## Обоснование

- UUID гарантирует глобальную уникальность без координации между сервисами.
- Исключает коллизии при миграции Project между группами (scope id меняется, UUID остаётся).
- Соответствует практике REST API: идентификаторы ресурсов не должны зависеть от их текущего местоположения.
- `ontology-service` ожидает стабильный идентификатор для адресации графа.

## Связанные ADR

- `ADR-DES.API.organization-rest-endpoints` — канонический REST-контракт; `ProjectDetail.ontology_id` и `OntologyDetail.project_id` должны быть UUID.
- `ADR-DES.SECURITY.gitlab-like-organization-model` — модель организации; Ontology идентифицируется по `ontology_id`, не по scope.
