# ADR-IMPL.STACK.port-mapping-strategy

**Дата:** 2026-05-19 (amended 2026-07-20, implemented 2026-07-24)
**Статус:** ACCEPTED

## Контекст

Для локальной разработки, CI и staging нужен стабильный и документированный port mapping всех сервисов VEDO Core. Без единой схемы порты назначаются фрагментарно, усложняют запуск окружения и повышают риск конфликтов.

Из анализа артефактов проекта установлено:
- В `ADR-DES.INFRA.monolith-vs-microservices` зафиксирована микросервисная декомпозиция и внутренние межсервисные вызовы через gRPC/protobuf.
- В `ADR-DES.API.protocol-stack-strategy` зафиксирован протокольный контракт: внутренний функциональный контур сервисов через gRPC, внешний контур через REST/GraphQL/WebSocket, API Gateway как единый фасад.
- В `ADR-IMPL.INTEGRATION.commenting-service-architecture` зафиксирован внешний WebSocket endpoint комментариев `wss://<host>/ws/comments` и REST-доступ через API Gateway.
- В `deploy/docker-compose.yml` уже используются публичные порты `3000` (frontend), `3002` (publish-browse-ui) и `8080` (API Gateway), а также dev-публикация инфраструктурных портов (`7474/7687`, `5432`, `6379`, `5672/15672`).
- Внутренние сервисы (`versioning-service`, `auth-service`, `metrics-service`, `publisher-service`, `public-browse-api`, `commenting-service`, `ticket-api`, `ticket-classifier`, `ticket-telemetry-listener`, `ticket-notifier`, `document-extractor`) ранее публиковались на host через `ports`. В процессе имплементации ADR (2026-07-24) переведены на `expose` в рамках bridge-сети Docker.
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

## Мультиокруженческий port mapping

Для поддержки параллельного запуска нескольких окружений (dev, test, staging) без конфликтов портов принята схема смещений:

| Окружение | Смещение | COMPOSE_PROJECT_NAME |
|-----------|----------|----------------------|
| Dev | нет (по умолчанию) | `vedo-core-dev` |
| Test | +10000 (RabbitMQ AMQP: +10001) | `vedo-core-test` |
| Staging | +20000 (RabbitMQ AMQP: +20002) | `vedo-core-staging` |

Соглашение об именовании переменных:
- `XXX_PORT` — хост-порт (меняется между окружениями)
- `XXX_CONTAINER_PORT` — контейнерный порт (одинаков во всех окружениях)
- `SERVICE_PORT` / `GRPC_PORT` внутри контейнера всегда используют контейнерные порты

### Application Services

| Сервис | Dev | Test (+10000) | Staging (+20000) |
|--------|-----|---------------|-------------------|
| Frontend (SPA) | 3000 | 13000 | 23000 |
| Publish Browse UI | 3002 | 13002 | 23002 |
| API Gateway | 8080 | 18080 | 28080 |
| Auth Service (gRPC) | 9003 | 19003 | 29003 |
| Versioning Service (gRPC) | 9002 | 19002 | 29002 |
| Metrics Service | 8084 | 18084 | 28084 |
| Publisher Service (gRPC) | 9005 | 19005 | 29005 |
| Public Browse API (gRPC) | 9011 | 19011 | 29011 |
| Commenting Service (gRPC) | 9004 | 19004 | 29004 |
| Ticket API (REST) | 8088 | 18088 | 28088 |
| Ticket API (gRPC) | 9010 | 19010 | 29010 |
| Ticket Classifier | 8089 | 18089 | 28089 |
| Ticket Telemetry Listener | 8090 | 18090 | 28090 |
| Ticket Notifier | 8091 | 18091 | 28091 |
| Document Extractor (gRPC) | 9013 | 19013 | 29013 |
| AI Orchestration (health) | 8093 | 18093 | 28093 |
| AI Orchestration (gRPC) | 9014 | 19014 | 29014 |

### Infrastructure

| Сервис | Dev | Test (+10000) | Staging (+20000) |
|--------|-----|---------------|-------------------|
| Neo4j (HTTP) | 7474 | 17474 | 27474 |
| Neo4j (Bolt) | 7687 | 17687 | 27687 |
| PostgreSQL | 5432 | 15432 | 25432 |
| Redis | 6379 | 16379 | 26379 |
| RabbitMQ (AMQP) | 5672 | 15673 | 25674 |
| RabbitMQ (Management) | 15672 | 25672 | 35672 |
| MinIO (API) | 9000 | 19000 | 29000 |
| MinIO (Console) | 9001 | 19001 | 29001 |
| Keycloak | 8180 | 18180 | 28180 |

### Observability (profile: `obs`)

| Сервис | Dev | Test (+10000) | Staging (+20000) |
|--------|-----|---------------|-------------------|
| Prometheus | 9090 | 19090 | 29090 |
| Grafana | 3001 | 13001 | 23001 |
| Loki | 3100 | 13100 | 23100 |
| Tempo (HTTP) | 3200 | 13200 | 23200 |
| Tempo (OTLP gRPC) | 4317 | 14317 | 24317 |
| Tempo (OTLP HTTP) | 4318 | 14318 | 24318 |
| OTEL Collector (Prometheus) | 8888 | 18888 | 28888 |
| OTEL Collector (Health) | 8889 | 18889 | 28889 |

### LLM (profile: `llm`)

| Сервис | Dev | Test (+10000) | Staging (+20000) |
|--------|-----|---------------|-------------------|
| Ollama | 11434 | 21434 | 31434 |

### Documentation (profile: `documentation`)

| Сервис | Dev | Test (+10000) | Staging (+20000) |
|--------|-----|---------------|-------------------|
| User Guide | 5000 | 15000 | 25000 |
| Developer Guide | 5001 | 15001 | 25001 |
| Admin Guide | 5002 | 15002 | 25002 |
| Integrator Guide | 5003 | 15003 | 25003 |

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
- Параллельный запуск dev/test/staging без конфликтов портов за счёт схемы смещений
- Единый справочник портов для всех окружений — dev, test (+10000), staging (+20000)

**Отрицательные:**
- Возможны конфликты с уже занятыми host-портами на машине разработчика
- Требуется миграция текущих compose-настроек внутренних сервисов на `expose` и обновление переменных сервисных URL

**Меры снижения рисков:**
- Поддержать переопределение host-портов через `.env` (`*_PORT`) без изменения container-port контрактов
- Вести единый справочник портов в `deploy/README.md` и Antora-документации (`admin-guide/deployment.adoc`)
- Проверять конфигурацию портов в CI (`docker compose config` + smoke-check endpoints)
- Фиксировать `COMPOSE_PROJECT_NAME` в каждом `.env.*` файле для изоляции namespace окружений

## Related ADRs

- `ADR-DES.INFRA.monolith-vs-microservices`
- `ADR-DES.API.protocol-stack-strategy`
- `ADR-IMPL.STACK.microservice-language-stack-strategy`
- `ADR-IMPL.INTEGRATION.commenting-service-architecture`
- `ADR-IMPL.OPS.ticket-management-system-architecture`
- `ADR-DES.INFRA.otel-observability-strategy`
- `ADR-IMPL.STACK.docker-adoption`
- `ADR-DES.PROCESS.deployment-strategy-policy`

---
