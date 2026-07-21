# ADR-DES.API.graphql-sparql-split-strategy: Использование раздельных API для навигации по графу и аналитических запросов

**Дата:** 2026-05-12  
**Статус:** Принято

## Контекст

VEDO Core предоставляет 2D/3D навигатор по онтологии. Фронтенду нужен API, который поддерживает пагинацию, запрос только необходимых полей, вложенные запросы и работу с динамической схемой онтологии.

Проблема онтологий в том, что пользовательские классы, свойства и отношения создаются во время работы системы и не могут быть полностью перечислены как статические поля GraphQL-схемы. Одновременно VEDO Core должен поддерживать сложные аналитические запросы к RDF/OWL-графу: фильтры, агрегации и обходы графа переменной глубины.

## Требование-источник
- [api-graphql-sparql.md](requirements/REQ-FUN.API.graphql-sparql.md)

## Решение

Использовать GraphQL как основной API для навигации по графу онтологии, а SPARQL точка доступа — для аналитических запросов.

GraphQL решает задачи навигации: cursor pagination, вложенные запросы, типизация и Apollo cache. Динамическая схема онтологии не ломает GraphQL, потому что пользовательские свойства возвращаются через контейнеры `propertyValues`, `outgoingEdges` и `incomingEdges`, а не как заранее сгенерированные поля схемы. SPARQL сохраняет полную выразительность W3C-стандарта для сложной аналитики, которая не покрывается навигационной схемой.

GraphQL-схему строить вокруг фиксированных интерфейсов `Entity`, `Class`, `Individual`, `Property`. Для защиты от DoS применить depth limit, query complexity, timeout, result cap и rate limiting. SPARQL точка доступа сделать read-only и отделить от GraphQL-маршрутов.

### Разделение ответственности (явная граница протоколов)

Граница между REST и GraphQL жёсткая и обязательная. Любое отклонение — архитектурное нарушение.

| Ответственность | Протокол | Метод | Эндпоинт/тип |
|-----------------|----------|--------|---------------|
| Чтение графа онтологии (навигация) | **GraphQL** | Query | `ontology`, `class`, `classes`, `classTree`, `classAncestors`, `classDescendants`, `graphNeighborhood`, `autocompleteClasses`, `property`, `properties`, `individual`, `individuals` |
| Чтение версии (commits, branches, tags) | **GraphQL** | Query | `commits`, `branch`, `branches`, `tags`, `compareRevisions` |
| Запись онтологии (CRUD классов, свойств, индивидов) | **REST** | POST/PUT/DELETE | `/api/v1/ontologies/{id}/classes`, `/api/v1/ontologies/{id}/properties`, `/api/v1/ontologies/{id}/individuals` |
| Запись онтологии (CRUD онтологий) | **REST** | POST/PUT/DELETE | `/api/v1/ontologies`, `/api/v1/ontologies/{id}` |
| Импорт/экспорт онтологии | **REST** | GET/POST | `/api/v1/ontologies/{id}/export`, `/api/v1/ontologies/{id}/import` |
| Выполнение SPARQL-запросов | **REST** | POST | `/api/v1/sparql` (с DoS-защитой через CircuitBreakerMiddleware) |
| Выполнение CYPHER-запросов | **REST** | POST | `/api/v1/cypher` (с DoS-защитой) |
| Версионирование операций (commit, branch, merge, rollback) | **REST** | POST/DELETE | `/api/v1/versioning/...` |
| Управление организацией (groups, projects, members, visibility, policies) | **REST** | CRUD | `/api/v1/groups`, `/api/v1/projects`, `/api/v1/ontologies/{id}/members` |
| Координация draft-состояния (dirty flag) | **REST** | POST/PUT | `/api/v1/ontologies/{id}/draft` (запланированный REST-эндпоинт) |
| Комментирование (вне графа онтологии) | **REST** | POST | (Эндпоинт rest через api-gateway, не через GraphQL mutations для онтологии) |
| SHACL-валидация | **REST** | POST | (Эндпоинт валидации через REST) |

**Запрещено в GraphQL (критично):**

1. **SPARQL/CYPHER-выполнение через GraphQL** — `sparqlQuery` и аналогичные GraphQL-резолверы **запрещены**. Они обходят DoS-защиту API Gateway (rate limiting, Circuit Breaker, query complexity checks в `CircuitBreakerMiddleware`, см. ADR-DES.API.sparql-dos-protection.md). SPARQL доступен только через REST `/api/v1/sparql`.
2. **GraphQL mutations для CRUD сущностей онтологии** — `createClass`, `createProperty`, `createIndividual`, `updateClass`, `deleteClass` и аналоги **запрещены** в GraphQL-схеме. Запись графа онтологии — только через REST. Это обеспечивает:
   - Единый путь idempotency (`Idempotency-Key` header на write-эндпоинтах, см. REQ-FUN.API.write-idempotency)
   - Прохождение через auth-мидлвэр API Gateway (OAuth2/JWT)
   - Аудит-лог (audit events) на write-операциях
   - Защиту от случайного обхода Circuit Breaker
3. **Динамическая регистрация пользовательских типов в GraphQL-схеме** — пользовательские классы/свойства возвращаются через контейнеры (`propertyValues`, `outgoingEdges`, `incomingEdges`), а не как новые поля схемы.

4. **Любые GraphQL mutations** — GraphQL-схема VEDO Core не содержит `Mutation` root. Все операции, изменяющие состояние (включая coordinate-draft, membership, draft-state, comments, validation), выполняются через REST. Узкие «временные» mutations (`updateDraft`, `updateMemberRole`, `removeMember`, существовавшие ранее) запрещены и подлежат/прошли миграции в REST (см. ADR-DES.API.rest-graphql-mutation-boundary.md).

## Рассмотренные альтернативы

**1. Универсальный SPARQL endpoint**
- Преимущество: гибкость и стандарт W3C для RDF.
- Проблема: offset/limit-пагинация неудобна для UI и может пропускать или дублировать записи при параллельных изменениях.
- Проблема: нет типизации на клиенте и выше порог входа для разработчиков клиентской части.
- Вердикт: оставить для аналитических запросов, но не использовать как основной API навигации.

**2. REST API с фиксированными endpoints**
- Преимущество: простота и широкая поддержка.
- Проблема: для получения подграфа требуется много запросов, возникает N+1.
- Проблема: плохо поддерживает произвольные пользовательские типы связей и выбор только нужных вложенных полей.
- Вердикт: использовать для внешних CRUD/integration scenarios, но не как основной API графовой навигации.

**3. Разделение GraphQL + SPARQL**
- Преимущество: GraphQL решает задачи навигации: пагинация, вложенные запросы, типизация, Apollo cache.
- Преимущество: SPARQL сохраняет полную выразительность для аналитики.
- Преимущество: динамическая схема онтологии не ломает GraphQL, потому что пользовательские свойства возвращаются через контейнеры.
- Вердикт: выбрано.

## Последствия

**Положительные последствия:**
- Клиентская часть получает API, удобный для 2D/3D навигации, ленивой подгрузки и cursor pagination.
- SPARQL остаётся доступным для опытных пользователей и аналитических сценариев.
- GraphQL-схема остаётся стабильной, несмотря на пользовательские свойства онтологии.

**Отрицательные последствия:**
- Нужно поддерживать два API-слоя с разными правилами документации, лимитов и диагностики.
- Клиент должен знать ID интересующих классов и связей через introspection или UI-контекст.
- GraphQL точка доступа требует контроля depth limit и query complexity.

**Меры снижения рисков:**
- Явно разделить документацию GraphQL-навигации и SPARQL-аналитики.
- Ввести обязательные лимиты: depth limit, complexity threshold, timeout, result cap и rate limiting.
- Логировать operation name, trace_id, redacted variables summary, result count и решение security checks для каждого GraphQL-запроса.

---
