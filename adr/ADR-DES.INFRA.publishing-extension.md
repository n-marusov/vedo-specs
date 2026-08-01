# ADR-DES.INFRA.publishing-extension

**Дата:** 2026-08-01  
**Статус:** Принято  
**Revision of:** [ADR-DES.INFRA.ontology-publishing.md](ADR-DES.INFRA.ontology-publishing.md)

## Контекст

Исходное ADR-DES.INFRA.ontology-publishing определило publisher-service + public-browse-api + второй read-only Neo4j для публичных снэпшотов. Однако текущая реализация — in-memory stubs: нет отдельного public Neo4j, нет rate limiting, publish-browse-ui — заглушка.

Анализ альтернатив (сессия 2026-08-01) выявил пробелы в модели:
- **Access control:** нет бинарной модели public/restricted (только «без авторизации»).
- **Visibility:** visibility снэпшота привязана к Project visibility — не всегда желательно.
- **Gate model:** нет явного publish gate; неясно, кто может публиковать.
- **Serving layer:** нет CQRS-разделения read-heavy serving от editing workspace.

## Требование-источник

- [REQ-CON.STACK.publishing-extension.md](../requirements/REQ-CON.STACK.publishing-extension.md)
- [REQ-CON.STACK.ontology-publishing.md](../requirements/REQ-CON.STACK.ontology-publishing.md)
- [ADR-DES.INFRA.ontology-publishing.md](ADR-DES.INFRA.ontology-publishing.md) — исходное решение

## Решение

**Расширить модель публикации: CQRS, бинарный access model, maintainer gate.**

- **CQRS:** снэпшот = материализованный read-only serving слой; чтения ≫ записи.
- **Serving store:** отдельное read-only хранилище (второй Neo4j или MinIO+index; решение отложено до реализации).
- **Access model (бинарный):**
  - *Public* — без auth, базовая защита от DoS (rate limiting).
  - *Restricted* — API key через заголовок `X-VEDO-API-Key`.
- **Publish gate:** Maintainer+ (вес роли ≥ 2 в `api-gateway/auth/auth.go RequiredRoleLevel`); отдельной роли publisher НЕТ. Owner (вес = 3) также может публиковать.
- **Snapshot visibility:** декоррелирована от Project visibility (приватный проект может публиковать публичный снэпшот).
- **Endpoint naming:** `/projects/{pid}/releases` (GitLab-нейминг; `snapshots` → `releases`).
- **Public browse shape:** TBD — плоский `/api/v1/public/releases/{id}` или вложенный `/api/v1/public/projects/{pid}/releases/{id}`.
- **Rate limiting:** per-IP и per-API-key; конфигурируемые лимиты.
- **API key management:** generate/revoke через `/projects/{pid}/api_keys` (post-MVP) или project settings.

### Фазы реализации

- **Phase 1 (M11):** snapshot publishing pipeline, serving store, базовый public browse API/UI.
- **Phase 2 (post-M11):** API key auth, rate limiting, продвинутые browse-функции.

### Анализ альтернатив хранения (storage alternatives)

| Вариант serving store | Плюсы | Минусы | Статус |
|-----------------------|-------|--------|--------|
| Второй read-only Neo4j | Единый графовый стек, Cypher | Операционные затраты, лицензирование | Кандидат |
| MinIO + index (Parquet/JSON) | Дешевле, масштабируется, object storage | Нужен отдельный index-слой | Кандидат |
| Read replica Neo4j Enterprise | Минимальная задержка | Требует Enterprise-лицензию | Отложено (cost analysis) |

## Рассмотренные альтернативы

| Альтернатива | Причина отклонения |
|--------------|--------------------|
| A) Исходная модель ADR (без CQRS, API key, visibility decoupling) | Недостаточна для реального publishing |
| B) Единый Neo4j с read replica (Enterprise) | Отложена для cost analysis |
| C) Расширенная модель с CQRS (выбрано) | Эволюционирует исходное решение без отмены |

## Последствия

**Положительные:**
- Чёткое разделение read-heavy serving от editing workspace.
- Бинарный access model покрывает реальные сценарии (public share + restricted partner access).
- Visibility декорреляция даёт гибкость публикации.
- Publish gate упрощает RBAC (нет отдельной роли).

**Отрицательные:**
- publisher-service требует полной переработки (не просто активация stubs).
- public-browse-api требует auth layer (API key validation).
- Операционные затраты второго serving store (Neo4j/MinIO).

**Меры снижения рисков:**
- Поэтапная реализация (Phase 1: pipeline + serving; Phase 2: auth + rate limiting).
- Решение по serving store отложено до имплементационной фазы с явным сравнением.
- API key management — post-MVP (project settings fallback).

## Связанные ADR

- [ADR-DES.INFRA.ontology-publishing.md](ADR-DES.INFRA.ontology-publishing.md) — исходное решение (revision of)
- [ADR-DES.API.rest-gitlab-alignment.md](ADR-DES.API.rest-gitlab-alignment.md) — `/projects/{pid}/releases` нейминг
- [ADR-DES.SECURITY.public-ontology-access.md](ADR-DES.SECURITY.public-ontology-access.md) — публичный периметр
- [REQ-CON.STACK.publishing-extension.md](../requirements/REQ-CON.STACK.publishing-extension.md)

---
