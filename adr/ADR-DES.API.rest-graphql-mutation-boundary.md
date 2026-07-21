# ADR-DES.API.rest-graphql-mutation-boundary: Жёсткая граница между REST (запись) и GraphQL (навигация)

**Дата:** 2026-07-21  
**Статус:** Принято

## Контекст

В текущей реализации VEDO Core обнаружены четыре архитектурных нарушения ADR-DES.API.graphql-sparql-split-strategy.md и ADR-DES.API.protocol-stack-strategy.md:

1. **GraphQL-мутации для CRUD сущностей онтологии.** В `src/services/frontend/src/apollo/queries.ts` объявлены `CREATE_CLASS_MUTATION`, `CREATE_PROPERTY_MUTATION`, `CREATE_INDIVIDUAL_MUTATION`, которые используются в Vue-компонентах (`CreateClassDialog.vue`, `CreatePropertyDialog.vue`, `CreateIndividualDialog.vue`). Эти мутации **не реализованы в Rust GraphQL-схеме** (`ontology-service/src/graphql/mutation.rs`) — они молча падают. ADR требует, чтобы запись графа онтологии шла только через REST.

2. **SPARQL через GraphQL** (`sparqlQuery` в `query.rs`). Обходит DoS-защиту API Gateway — `CircuitBreakerMiddleware`, rate limiting, query complexity checks (см. ADR-DES.API.sparql-dos-protection.md § «Интеграция с API Gateway»). Безопасный путь — REST `/api/v1/sparql`, у которого эта защита есть.

3. **Неполный OpenAPI.** В `src/services/api-gateway/docs/openapi.json` отсутствуют 15+ write-эндпоинтов онтологии (POST/PUT/DELETE для ontologies, classes, properties, individuals, export/import), эндпоинты версионирования и AI. AI-агенты не могут корректно понять, какие REST-операции поддерживаются.

4. **Неявные ADR-ы.** До настоящего обновления ADR-ы говорили «GraphQL — для навигации», но не формулировали явного запрета на мутации и не описывали миграционный путь.

## Требование-источник

- [REQ-FUN.API.graphql-sparql.md](requirements/REQ-FUN.API.graphql-sparql.md)
- [ADR-DES.API.graphql-sparql-split-strategy.md](ADR-DES.API.graphql-sparql-split-strategy.md)
- [ADR-DES.API.protocol-stack-strategy.md](ADR-DES.API.protocol-stack-strategy.md)
- [ADR-DES.API.sparql-dos-protection.md](ADR-DES.API.sparql-dos-protection.md)

## Решение

Установить **жёсткую, обязательную границу** протоколов:

```
+---------------------------------------------------------------+
|                  API Gateway (единый фасад)                   |
+--------------------------+------------------------------------+
|           REST           |            GraphQL                 |
+--------------------------+------------------------------------+
| Все записи вграф         | Только read-only навигация         |
| онтологии (CRUD онтоло-  | по графу (Class, Property,         |
| гий, классов, свойств,   | Individual, branches, commits,    |
| индивидов)               | tags, neighborhood)               |
|                          |                                    |
| Импорт/экспорт           | Запрещено:                         |
| SPARQL/CYPHER выполнение |  - sparqlQuery GraphQL resolver    |
| Версия (commit/branch/   |  - createClass, createProperty,   |
|  merge/rollback)         |    createIndividual mutations      |
| Управление организацией  |  - updateClass, deleteClass и т.п.|
| Валидация (SHACL)        |  - updateDraft, updateMemberRole, |
| Комментарии              |    removeMember (мigrate -> REST) |
| Координация draft-state   |                                    |
+--------------------------+------------------------------------+
|                     gRPC внутрь сервисов                      |
+---------------------------------------------------------------+
```

### Почему GraphQL-мутации для сущностей онтологии запрещены

1. **Обход Idempotency-Key.** REST write-эндпоинты требуют `Idempotency-Key` header (REQ-FUN.API.write-idempotency). GraphQL mutations не имеют единого канала для HTTP-header-контроля — это привело бы к дублированию или потере idempotency.
2. **Обход auth-мидлвэра.** Auth middleware API Gateway сидит на `/api/v1/*`. GraphQL passthrough в routes.go (`api.Any("/graphql", ...)`) наследует middleware, но CircuitBreaker и DoS-филtreы, специфичные для `/sparql`, к GraphQL не применяются.
3. **Аудит-лог write-операций.** Аудит событий организован на уровне REST handlers (`handlers/*.go`), GraphQL mutations проходят мимо этого слоя.
4. **Согласованность спецификаций.** OpenAPI остаётся единственным публичным контрактом для writes. Дублирование того же поведения в GraphQL-схеме усложняет интеграцию и тестирование.

### Почему SPARQL через GraphQL запрещён

1. **Обход CircuitBreakerMiddleware** (ADR-DES.API.sparql-dos-protection.md § 6 «Интеграция с API Gateway»). Middleware навешан на `POST /api/v1/sparql`, но не на `/api/v1/graphql`. GraphQL-путь уходит мимо circuit breaker, rate limit и filter.
2. **Безфильтрованная query complexity.** Pre-filter уровня 1 (блокировка опасных паттернов) применяется только в REST-handler `queryHandler.HandleSPARQL`. GraphQL-резолвер `sparqlQuery` в `query.rs` таких проверок не делает.
3. **监控ing и alerting.** Monitoring-level 3 ADR завязан на метрики REST `/api/v1/sparql` (`slo_sparql_rps`, `sparql_errors_total`). GraphQL-путь не инжектирует нужные метки в trace context.

### Миграционный путь

#### Фронтенд — компоненты, требующие миграции

| Vue-компонент | Текущая (запрещена) реализация | Целевая REST-реализация |
|---------------|-------------------------------|--------------------------|
| `CreateClassDialog.vue` | `useMutation(CREATE_CLASS_MUTATION)` | `axios.post('/api/v1/ontologies/{id}/classes', payload)` |
| `CreatePropertyDialog.vue` | `useMutation(CREATE_PROPERTY_MUTATION)` | `axios.post('/api/v1/ontologies/{id}/properties', payload)` |
| `CreateIndividualDialog.vue` | `useMutation(CREATE_INDIVIDUAL_MUTATION)` | `axios.post('/api/v1/ontologies/{id}/individuals', payload)` |
| `SPARQLPage.vue` | Apollo `SPARQL_EXECUTE_QUERY` (GraphQL query `sparqlQuery`) | `axios.post('/api/v1/sparql', { ontologyId, query, limit, offset })` |

REST-эндпоинты сами уже есть в `routes.go` (строки 122-135), фронтенду нужно только переключиться на axios.

#### Удаление из GraphQL-схемы

1. `sparqlQuery` — удалить из `ontology-service/src/graphql/query.rs`.
2. `CREATE_*_MUTATION` экспорты — удалить из `frontend/src/apollo/queries.ts`.
3. `SPARQL_EXECUTE_QUERY` — удалить из `frontend/src/apollo/queries.ts`.
4. `updateDraft`, `updateMemberRole`, `removeMember` — удалить из `MutationRoot` в `ontology-service/src/graphql/mutation.rs`. Мигрировать в REST-эндпоинты (см. план ниже). GraphQL `Mutation` root в итоговой схеме отсутствует.

#### Миграция оставшихся GraphQL mutations в REST

| Мутация | Целевой REST-эндпоинт | Обоснование |
|---------|-------------------------|-------------|
| `updateDraft(ontologyId, changes)` | `PUT /api/v1/ontologies/{id}/draft` | Координация dirty-state — write-операция на workspace, должна попадать под auth/audit/idempotency |
| `updateMemberRole(ontologyId, userId, role)` | `PUT /api/v1/ontologies/{id}/members/{userId}` | Управление членством — запись в орг. модель, REST эндпоинт уже существует в routes.go |
| `removeMember(ontologyId, userId)` | `DELETE /api/v1/ontologies/{id}/members/{userId}` | Удаление членства — REST эндпоинт уже существует в routes.go |

После миграции `MutationRoot` в Rust помечается `EmptyMutation` (или удаляется из `Schema::build`).

## Рассмотренные альтернативы

**1. Разрешить GraphQL-мутации для записи графа онтологии.**  
Отклонено — противоречит требованиям auth/idempotency/audit, дублирует REST, усложняет спецификацию и DoS-защиту.

**2. Носить DoS-защиту в сам GraphQL-резолвер SPARQL.**  
Отклонено — дублирование middleware, рассредоточение защиты по слоям, расхождение с ADR-DES.API.sparql-dos-protection.md § 6.

**3. Полный запрет всех GraphQL-mutations.**  
Принято. `updateDraft` (draft-state coordination) и `updateMemberRole`/`removeMember` (org management) ранее существовали как «узкие» mutations, но нарушали общий принцип: GraphQL не должен делать write-операций, потому что обходит auth-middleware для writes, audit-log, Idempotency-Key и CircuitBreaker. Они мигрированы в REST (см. § «Миграционный путь»). `Mutation` root удаляется из GraphQL-схемы полностью.

## Последствия

**Положительные:**
- Единый путь записи через REST — единый auth, idempotency, audit.
- SPARQL выполняется только через защищённый REST — полная DoS-защита.
- OpenAPI становится единственным контрактом для writes — AI-агенты и интеграторы могут корректно понять API.
- Меньше кода в GraphQL-схеме Rust — меньше поверхность атаки и сложность.

**Отрицательные:**
- 4 Vue-компонента требуют миграции на REST (небольшое усилие, эндпоинты уже есть).
- `sparqlQuery` удалён — клиенты должны использовать REST `/api/v1/sparql`.
- Frontend-test fixtures для SPARQL (`MOCK_SPARQL_RESULTS` в `mock-data.ts`) становятся неиспользуемыми — их нужно убрать.

**Меры снижения рисков:**
- REST-эндпоинты уже существуют в `routes.go` — миграция фронтенда сводится к замене `useMutation` на `axios.post`.
- Дополнительно — интеграционные тесты покрывают новые REST-вызовы (см. Plan task 9).
- Документация GraphQL-схемы выведена в отдельный файл `src/services/api-gateway/docs/graphql-schema.md` — больше не смешивается с OpenAPI.

## Связанные ADR

- [ADR-DES.API.graphql-sparql-split-strategy.md](ADR-DES.API.graphql-sparql-split-strategy.md) — § «Разделение ответственности (явная граница протоколов)» обновлён взаимно.
- [ADR-DES.API.protocol-stack-strategy.md](ADR-DES.API.protocol-stack-strategy.md) — § «Разделение ответственности протоколов» обновлён взаимно.
- [ADR-DES.API.sparql-dos-protection.md](ADR-DES.API.sparql-dos-protection.md) — § 6 «Интеграция с API Gateway»: middleware навешан на REST, не на GraphQL.

## Чек-лист реализации

- [ ] `sparqlQuery` удалён из `ontology-service/src/graphql/query.rs`
- [ ] `SPARQL_EXECUTE_QUERY` удалён из `frontend/src/apollo/queries.ts`
- [ ] `CREATE_CLASS_MUTATION`, `CREATE_PROPERTY_MUTATION`, `CREATE_INDIVIDUAL_MUTATION` удалены из `queries.ts`
- [ ] `updateDraft`, `updateMemberRole`, `removeMember` удалены из `MutationRoot` в `mutation.rs` и мигрированы в REST
- [ ] `Mutation` root удалён из GraphQL-схемы (используется `EmptyMutation`)
- [ ] `CreateClassDialog.vue` / `CreatePropertyDialog.vue` / `CreateIndividualDialog.vue` переведены на REST
- [ ] `SPARQLPage.vue` переведён на REST `/api/v1/sparql`
- [ ] OpenAPI (`openapi.json`) содержит все write-эндпоинты онтологии + версионирования
- [ ] Существует отдельная GraphQL-схема-документация (`graphql-schema.md`)
- [ ] Интеграционный тест: REST entity CRUD возвращает 200/201, требует auth
- [ ] Интеграционный тест: GraphQL-интроспекция не содержит `sparqlQuery` и entity mutations

---