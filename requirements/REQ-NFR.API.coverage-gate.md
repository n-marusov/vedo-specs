# API Coverage Gate: Customer-Facing API Classification v1.0

| Атрибут | Значение |
|---------|----------|
| **ID** | REQ-NFR.API.coverage-gate |
| **Уровень** | NFR |
| **Атрибут качества** | Functionality |
| **Приоритет** | P1 |
| **Статус** | УТВЕРЖДЕНО |
| **Источник** | — |
| **Критерии приёмки** | — |

---

## Что описывает

Документ фиксирует границу между customer-facing и internal API поверхностями VEDO Core, определяет SLA/compatibility commitments по каждой поверхности и правила проверки соответствия в CI.

Цель: устранить неоднозначность, какие API покрываются продуктными и эксплуатационными гарантиями, а какие являются внутренними контрактами без customer-facing обязательств.

## 1) Определение customer-facing API

Customer-facing API в контексте VEDO Core — это API поверхность, одновременно соответствующая критериям:
- доступна внешним клиентам/интеграциям или UI-клиенту как часть публичного продукта;
- документирована как поддерживаемый интерфейс для заказчика;
- подпадает под SLA наблюдаемости и инцидентного реагирования;
- имеет формальные compatibility commitments (deprecation, breaking-change policy, versioning rules).

Если API не удовлетворяет этим критериям, он классифицируется как internal и не покрывается SLA/compatibility гарантиями для клиентов.

## 2) Классификация API surfaces

| API surface | Customer-facing | SLA coverage | Compatibility commitment | Обоснование |
|---|---|---|---|---|
| REST (`/api/v1/*`) | Yes | Yes | Yes | Основной публичный контракт для интеграций и UI; фиксированная версия в URL |
| GraphQL (`/graphql`) | Yes | Yes | Yes | Публичный навигационный API; эволюция через `@deprecated` |
| SPARQL (`/sparql`) | Yes | Yes | Yes | Публичный аналитический API; W3C-совместимость, MVP: `SELECT/ASK` |
| Public Browse API (`/browse/*`) | Yes | Yes | Yes | Публичный read-only сценарий просмотра онтологий |
| WebSocket (public realtime channels) | Yes | Yes | Yes | Публичный протокол коллаборации; версионируемый handshake |
| Admin API (`/api/v1/admin/*`) | Partial | Partial | Partial | Доступен ограниченному классу клиентов (tenant admins); best-effort на части операций |
| gRPC internal services | No | No | No | Внутренний межсервисный контракт, может меняться без customer notice |
| Internal REST (`/health`, `/ready`, `/`) | No | No | No | Management surface, не предназначен для клиентской интеграции |
| Internal gRPC reflection | No | No | No | Инструмент разработки/диагностики, не публичный контракт |

## 3) Boundary diagram: customer-facing vs internal

```mermaid
flowchart LR
    subgraph Client_Zone[Client Zone]
      UI[VEDO Web UI]
      EXT[External Integrations]
    end

    subgraph Customer_Facing_API[Customer-Facing API Boundary]
      REST[/REST /api/v1/*/]
      GQL[/GraphQL /graphql/]
      SPQ[/SPARQL /sparql/]
      BRW[/Browse /browse/*/]
      WS[/WebSocket public/]
      ADM[/Admin API /api/v1/admin/* (partial)/]
    end

    subgraph Internal_API[Internal API Boundary]
      GRPC[gRPC Internal]
      MNG[/health /ready /]
      REFL[gRPC reflection]
    end

    UI --> REST
    UI --> GQL
    UI --> SPQ
    UI --> BRW
    UI --> WS
    EXT --> REST
    EXT --> GQL
    EXT --> SPQ
    EXT --> BRW
    EXT --> ADM

    REST --> GRPC
    GQL --> GRPC
    SPQ --> GRPC
    WS --> GRPC
```

## 4) Compatibility commitments by API surface

| API surface | Deprecation period | Versioning strategy | Breaking change policy | Testing requirements (coverage gate) |
|---|---|---|---|---|
| REST | 12 months | URL versioning (`/v1/`, `/v2/`) | Только через major version increment | OpenAPI diff + contract tests + backward compatibility tests |
| GraphQL | 9 months (`@deprecated`) | Без version URL; schema evolution | Additive only; удаление только после deprecation window | Schema diff + deprecated usage checks + integration tests |
| SPARQL | N/A (W3C standard) | Версия языка по стандарту W3C | В MVP breaking changes не допускаются | Query compatibility suite (SELECT/ASK) + response contract checks |
| WebSocket | 6 months | Handshake version header | Backward-compatible protocol required | Protocol compatibility tests + reconnect tests |
| Public Browse API | 12 months | Как у REST | Как у REST | OpenAPI diff + browse flow integration tests |
| Admin API | Partial, best effort | Как у REST | Допускаются изменения с обязательным release note | Contract smoke tests + tenant admin scenario tests |

## 5) Исключения (не covered)

Не покрываются customer-facing SLA/compatibility commitments:
- internal gRPC контракты между сервисами;
- internal management endpoints (`/health`, `/ready`, `/`);
- internal gRPC reflection;
- experimental API, помеченные `X-VEDO-Experimental: true`;
- endpoints/features в alpha/beta статусе, явно помеченные в документации;
- deprecated endpoints после даты `Sunset` (end-of-life).

## 6) SLA Commitments by API Surface (для публичной документации)

| API surface | SLA monitoring | Availability commitment scope | Compatibility scope | Customer notice requirement |
|---|---|---|---|---|
| REST | Да | Полный публичный path `/api/v1/*` | Полный | Да, release notes + deprecation headers |
| GraphQL | Да | Публичный endpoint `/graphql` | Полный | Да, `@deprecated` и changelog схемы |
| SPARQL | Да | Публичный endpoint `/sparql` | Полный в рамках поддерживаемого подмножества MVP | Да, release notes |
| Public Browse | Да | Публичный просмотр `/browse/*` | Полный | Да |
| WebSocket | Да | Публичные realtime-сессии | Полный | Да, protocol version notes |
| Admin API | Частично | Только поддерживаемые tenant-admin операции | Частично | Да, best-effort уведомления |
| Internal APIs | Нет | Не применимо | Не применимо | Нет |

## 7) Enforcement: как проверяется соблюдение

### 7.1 API contract tests
- REST/Public Browse: обязательные контрактные тесты против `openapi.yaml`.
- GraphQL: schema compatibility tests (breaking field/type changes), execution tests.
- SPARQL: совместимость запросов (MVP `SELECT/ASK`), проверка формата ответа.
- WebSocket: protocol handshake/version tests, backward compatibility на уровне событий.

### 7.2 Breaking change detection в CI
- REST/OpenAPI: автоматический diff между baseline и MR (блокировка на breaking changes без major bump).
- GraphQL: schema diff (блокировка удаления/изменения без deprecation window).
- WebSocket: protocol contract diff (блокировка несовместимого handshake/event schema).
- SPARQL: regression-suite для поддерживаемых конструкций языка.

### 7.3 Deprecation header validation
- REST/Public Browse/Admin: обязательные `Warning` + `Sunset` заголовки на deprecated endpoints.
- Проверка в CI: endpoint с deprecation-статусом без корректных заголовков => fail.
- Проверка сроков: endpoint после `Sunset` не может оставаться в списке active commitments.

## 8) Coverage gate rules

Gate считается пройденным только если одновременно:
- все customer-facing API surfaces имеют валидные contract tests;
- не обнаружены недекларированные breaking changes;
- deprecation policy соблюдена по каждому surface;
- operation inventory покрывает customer-facing операции UI;
- для partial APIs (Admin) отмечен статус best effort и документированы исключения.

Blocking conditions (MR fail):
- breaking change в REST/GraphQL/WebSocket без разрешённой стратегии миграции;
- удаление customer-facing endpoint/field без deprecation window;
- отсутствие обязательных `Sunset`/`Warning` заголовков для deprecated REST surfaces;
- несоответствие operation inventory и customer-facing surface mapping.

## 9) Операционные и продуктовые обязанности

| Роль | Ответственность |
|---|---|
| Product Manager | Утверждение customer-facing классификации и compatibility windows |
| API Architect | Политики versioning и breaking-change governance |
| Engineering Lead | Реализация contract tests и CI enforcement |
| Support Lead | Коммуникация deprecation/Sunset клиентам |
| SRE | SLA monitoring по customer-facing surfaces |

## 10) Статус

Решено.
