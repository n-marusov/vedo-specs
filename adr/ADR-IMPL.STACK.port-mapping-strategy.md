# ADR-IMPL.STACK.port-mapping-strategy

**Дата:** 2026-05-19
**Статус:** PROPOSED

## Контекст

Для локальной разработки, CI и staging нужен стабильный и документированный port mapping всех сервисов VEDO Core. Без единой схемы порты назначаются фрагментарно, усложняют запуск окружения и повышают риск конфликтов.

Из анализа артефактов проекта установлено:
- В `ADR-DES.INFRA.monolith-vs-microservices` зафиксирована микросервисная декомпозиция и внутренние межсервисные вызовы через gRPC/protobuf.
- В `ADR-DES.API.protocol-stack-strategy` зафиксирован протокольный контракт: внутренний функциональный контур сервисов через gRPC, внешний контур через REST/GraphQL/WebSocket, API Gateway как единый фасад.
- В `ADR-IMPL.INTEGRATION.commenting-service-architecture` зафиксирован внешний WebSocket endpoint комментариев `wss://<host>/ws/comments` и REST-доступ через API Gateway.
- В `deploy/docker-compose.yml` уже используются публичные порты `3000` (frontend) и `8080` (API Gateway), а также dev-публикация инфраструктурных портов (`7474/7687`, `5432`, `6379`, `5672/15672`).
- В `deploy/docker-compose.observability.yml` уже используются порты observability: `9090` (Prometheus), `3100` (Loki), `3200` (Tempo), `4317/4318` (OTLP), `8888/8889` (otel-collector); Grafana опубликована как `3001:3000` из-за занятости `3000` фронтендом.
- В `deploy/README.md` зафиксированы пользовательские endpoints (`http://localhost:3000`, `http://localhost:8080/health`, `http://localhost:3001`, `http://localhost:9090`).
- В `requirements/REQ-CON.INFRA.deployment-strategy.md` требования к конкретным номерам портов не найдены (документ про rollout-стратегии).
- В `requirements/REQ-NFR.DATA.backup-retention.md` требования к портам не найдены (документ про retention/SLA purge).

## Решение

Принять единую стратегию портов для Docker Compose контуров (local dev, CI, staging):
- host port равен container port по умолчанию;
- исключение допускается только при документированном конфликте (как для Grafana `3001:3000`);
- внешне публикуются только публичные входные точки и observability/dev-инфраструктура;
- внутренние service-to-service порты публикуются только через `expose` (без `ports`) в bridge-сети Compose.

**Публичные сервисы (доступны с хоста):**

| Сервис | Порт | Протокол | Обоснование |
|--------|------|----------|-------------|
| Frontend (Vue SPA) | 3000 | HTTP | Уже используется в `deploy/docker-compose.yml` и `deploy/README.md`; dev-вход в SPA |
| API Gateway | 8080 | HTTP (REST/GraphQL/SPARQL) | По `ADR-DES.API.protocol-stack-strategy` внешний фасад системы; уже используется в `deploy/docker-compose.yml` |
| Public Browse API | 8080 (через API Gateway) | HTTP/GraphQL | По `ADR-DES.API.protocol-stack-strategy` внешний API проходит через Gateway; отдельный host-порт не назначается |
| Commenting WebSocket | 8080 (через API Gateway, путь `/ws/comments`) | WebSocket | По `ADR-IMPL.INTEGRATION.commenting-service-architecture` внешний WS endpoint и по `ADR-DES.API.protocol-stack-strategy` внешний контур через Gateway |

**Внутренние сервисы (НЕ публикуются на хост, только `expose`):**

| Сервис | Порт | Протокол | Обоснование |
|--------|------|----------|-------------|
| Ontology Service | 9001 | gRPC | По `ADR-DES.API.protocol-stack-strategy` внутренний функциональный контур gRPC |
| Versioning Service | 9002 | gRPC | По `ADR-DES.API.protocol-stack-strategy` внутренний функциональный контур gRPC |
| Auth Service | 9003 | gRPC | По `ADR-DES.API.protocol-stack-strategy` внутренний функциональный контур gRPC |
| Commenting Service (internal API) | 9004 | gRPC | Внешний REST/WS идёт через Gateway, межсервисные вызовы стандартизируются по внутреннему gRPC-контуру |
| Publisher Service | 9005 | gRPC | CPU-bound backend-сервис из `ADR-IMPL.STACK.microservice-language-stack-strategy`, внутренний вызов через Gateway |
| Ticket Service | 9010 | HTTP | По `ADR-IMPL.OPS.ticket-management-system-architecture` Ticket API реализован как REST, в Compose-контуре остаётся internal-only |
| Public Browse Service | 9011 | gRPC | Публичный доступ через Gateway, внутренний read-only контур через gRPC |
| Metrics Service | 9012 | HTTP | По `ADR-IMPL.STACK.metrics-python-strategy` и текущему HTTP профилю FastAPI; внешняя публикация не требуется |

Примечание: management surface (`/health`, `/ready`) внутренних сервисов остаётся локальным для контейнера/сети Compose и не открывается на host.

**Базы данных и кэши (опционально, только dev-профиль):**

| Сервис | Порт | Протокол | Условие публикации |
|--------|------|----------|---------------------|
| Neo4j Browser | 7474 | HTTP | Только dev/debug, уже используется в `deploy/docker-compose.yml` |
| Neo4j Bolt | 7687 | Bolt | Только dev/debug, уже используется в `deploy/docker-compose.yml` |
| PostgreSQL | 5432 | PostgreSQL | Только dev/debug, уже используется в `deploy/docker-compose.yml` |
| Redis | 6379 | RESP | Только dev/debug, уже используется в `deploy/docker-compose.yml` |
| RabbitMQ AMQP | 5672 | AMQP | Только dev/debug, уже используется в `deploy/docker-compose.yml` |
| RabbitMQ Management | 15672 | HTTP | Только dev/debug, уже используется в `deploy/docker-compose.yml` |

**Observability (опционально, профиль `obs`):**

| Сервис | Порт | Протокол | Обоснование |
|--------|------|----------|-------------|
| Prometheus | 9090 | HTTP | Уже используется в `deploy/docker-compose.observability.yml` |
| Grafana | 3001 (host) / 3000 (container) | HTTP | Уже используется в `deploy/docker-compose.observability.yml`; исключение из host==container из-за конфликта с frontend:3000 |
| Loki | 3100 | HTTP | Уже используется в `deploy/docker-compose.observability.yml` |
| Tempo | 3200 | HTTP | Уже используется в `deploy/docker-compose.observability.yml` |
| Tempo OTLP gRPC | 4317 | gRPC | Уже используется в `deploy/docker-compose.observability.yml` |
| Tempo OTLP HTTP | 4318 | HTTP | Уже используется в `deploy/docker-compose.observability.yml` |
| OpenTelemetry Collector metrics | 8888 / 8889 | HTTP | Уже используется в `deploy/docker-compose.observability.yml` |

## Рассмотренные альтернативы

| Альтернатива | Причина отклонения |
|--------------|--------------------|
| Публиковать все сервисы на host (включая внутренние) | Нарушает принцип API Gateway как единого фасада (`ADR-DES.API.protocol-stack-strategy`), ухудшает безопасность и увеличивает риск коллизий портов |
| Оставить нефиксированные/случайные порты по сервисам | Усложняет запуск dev/CI/staging, документацию и диагностику; повышает риск drift между окружениями |
| Делать отдельные схемы портов для dev и CI | Увеличивает когнитивную нагрузку и стоимость поддержки; приоритет у единой воспроизводимой схемы |

## Последствия

**Положительные:**
- Предсказуемый запуск локального окружения и CI/staging без ручного подбора портов
- Консистентность с API-архитектурой: внешний трафик через API Gateway, внутренний через internal-only порты
- Упрощение диагностики и документации окружения

**Отрицательные:**
- Возможны конфликты с уже занятыми host-портами на машине разработчика
- Требуется миграция текущих compose-настроек внутренних сервисов на `expose` и обновление переменных сервисных URL

**Меры снижения рисков:**
- Поддержать переопределение host-портов через `.env` (`*_PORT`) без изменения container-port контрактов
- Вести единый справочник портов в `deploy/README.md` и `docs/ports.md`
- Проверять конфигурацию портов в CI (`docker compose config` + smoke-check endpoints)

## Related ADRs

- `ADR-DES.INFRA.monolith-vs-microservices`
- `ADR-DES.API.protocol-stack-strategy`
- `ADR-IMPL.STACK.microservice-language-stack-strategy`
- `ADR-IMPL.INTEGRATION.commenting-service-architecture`
- `ADR-IMPL.OPS.ticket-management-system-architecture`
- `ADR-DES.INFRA.otel-observability-strategy`

---
