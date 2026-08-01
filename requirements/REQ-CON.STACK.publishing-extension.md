# Расширение модели публикации онтологий (CQRS, access model, publish gate)

| Атрибут | Значение |
|---------|----------|
| **ID** | REQ-CON.STACK.publishing-extension |
| **Уровень** | CON |
| **Атрибут качества** | Functionality |
| **Приоритет** | P1 |
| **Статус** | ЧЕРНОВИК |
| **Источник** | Исследовательская сессия 2026-08-01: publishing model (CQRS, binary visibility, maintainer gate); ревизия REQ-CON.STACK.ontology-publishing |

---

## Назначение

Расширить базовое требование публикации онтологий ([REQ-CON.STACK.ontology-publishing](REQ-CON.STACK.ontology-publishing.md)) моделью CQRS, бинарной моделью доступа и maintainer-гейтом публикации. Данное требование является ревизией (revision) исходного требования публикации и отменяет его отдельные положения в части архитектуры serving layer и доступа.

## Требование

**Snapshot-модель (CQRS):**

- Снимок (snapshot / release) — материализованный read-only serving слой онтологии на конкретном коммите.
- Чтения ≫ записи: serving слой отделён от editing workspace.
- Serving store — отдельное read-only хранилище (второй Neo4j или MinIO+index; решение отложено до реализации, см. ADR-DES.INFRA.publishing-extension).

**Модель доступа (бинарная):**

- **Public** — без аутентификации; базовая защита от DoS (rate limiting).
- **Restricted** — аутентификация через API key (заголовок `X-VEDO-API-Key`); ключ привязан к конкретному release.

**Publish gate:**

- Публикацию выполняет роль **Maintainer+** (вес роли ≥ 2), без отдельной роли publisher.
- Owner (вес = 3) также может публиковать.

**Visibility:**

- Visibility снимка НЕ привязана к visibility Project: приватный проект может публиковать публичный снимок.

**Именование:**

- Эндпоинты: `/projects/{pid}/releases` (GitLab-нейминг; `snapshots` → `releases`).

**Rate limiting:**

- Per-IP и per-API-key; конфигурируемые лимиты.

**Управление ключами:**

- Генерация/отзыв ключей через `/projects/{pid}/api_keys` (post-MVP) или project settings.

## Ревизия исходного требования

| Положение REQ-CON.STACK.ontology-publishing | Ревизия |
|---------------------------------------------|---------|
| Read-only просмотр без аутентификации | Сохранено для public снимков; restricted требует API key |
| 2D/3D навигация | Сохранено |
| Полнотекстовый поиск | Сохранено |
| Полная изоляция от рабочей онтологии | Сохранено; усилено CQRS serving store |
| Отдельный read-only инстанс Neo4j | Решение отложено: второй Neo4j ИЛИ MinIO+index |
| Нет интеграции с внешними системами | Сохранено (интеграция через платформенный API) |

## Фазы реализации

- **Phase 1 (M11):** snapshot publishing pipeline, serving store, базовый public browse API/UI.
- **Phase 2 (post-M11):** API key auth, rate limiting, продвинутый browse.

## Критерии приёмки

1. Снимки материализуются в отдельный read-only serving store.
2. Public снимки доступны без аутентификации; restricted — только с валидным API key.
3. Публикация доступна Maintainer+ (вес ≥ 2).
4. Visibility снимка декоррелирована от visibility Project.
5. Снимки экспонируются как `/projects/{pid}/releases`.

## Ссылки (References)

- [REQ-CON.STACK.ontology-publishing.md](REQ-CON.STACK.ontology-publishing.md) — исходное требование публикации
- [ADR-DES.INFRA.publishing-extension.md](../adr/ADR-DES.INFRA.publishing-extension.md) — архитектурное решение-расширение
- [ADR-DES.INFRA.ontology-publishing.md](../adr/ADR-DES.INFRA.ontology-publishing.md) — исходное ADR публикации
- [ADR-DES.SECURITY.public-ontology-access.md](../adr/ADR-DES.SECURITY.public-ontology-access.md) — публичный периметр доступа

---

## Обоснование (Rationale)

Исходное ADR-DES.INFRA.ontology-publishing не реализовано (publisher-service + public-browse-api = in-memory stubs, нет отдельного public Neo4j, нет rate limiting). Анализ альтернатив (сессия 2026-08-01) выявил пробелы в access control, visibility и gate model. Данная ревизия эволюционирует исходное решение без его отмены: добавляет CQRS, бинарную модель доступа и maintainer-гейт.
