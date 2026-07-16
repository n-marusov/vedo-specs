---
title: "VEDO Core Technology Stack"
status: "approved"
date: "2026-05-09"
deciders: ["VEDO Core Team"]
---

# Технологический стек VEDO Core

## Контекст

VEDO Core — это высокопроизводительная веб-платформа для многопользовательского редактирования онтологий (графов знаний). Система должна обеспечивать:

- Отклик интерфейса < 1 сек (p95) при работе с онтологиями до 1 млн узлов; открытие класса с 1000 дочерних элементов за < 1 сек
- Параллельная работа через ветки с семантическим слиянием и Merge Request'ами
- Git-подобное версионирование с семантическим слиянием
- REST/gRPC API для интеграции с отраслевыми решениями
- Надёжное логирование, трассировку и мониторинг

Архитектурно система разделена на:
- **Frontend** (Web UI)
- **Backend** (микросервисы на Rust, Go, Python)
- **Слой данных** (Neo4j, PostgreSQL, Redis)
- **Стек наблюдаемости** (OpenTelemetry, Grafana)

## Решения

### 1. Frontend: Vue 3

**Решение:** Использовать Vue 3 с Composition API, TypeScript, Apollo Client (GraphQL), pnpm для управления зависимостями, Biome для линтинга и форматирования TypeScript/TSX-кода, и Pencil.dev для дизайна.

**Alternatives considered:**
- **React 18** — более высокая сложность обучения, boilerplate-код, необходимость выбора из множества библиотек.
- **Svelte 4** — меньшая экосистема, меньше готовых компонентов для графов.
- **Angular 16** — избыточен для нашего use case, тяжеловесен.

**Consequences:**
- ✅ Производительность: Virtual DOM с реактивностью на уровне компиляции, повторный рендер только изменённых узлов; целевой frame budget ≤ 16 мс для CRUD-операций
- ✅ Onboarding: новый разработчик поднимает локальный стек и выполняет базовый RDF/import сценарий за ≤ 60 минут (developer runbook)
- ✅ Отличная TypeScript поддержка
- ✅ Экосистема: Vite для быстрой сборки, pnpm для быстрой установки зависимостей, Apollo Client для управления состоянием
- ✅ Vue Flow для визуализации графов (альтернатива React Flow)

**Используемые особенности Vue 3:**
- Composition API для повторного использования логики
- Teleport для модальных окон и уведомлений
- Suspense для асинхронных компонентов
- `<script setup>` синтаксис для чистоты кода

**Управление состоянием:** Apollo Client (GraphQL-first, встроенный кэш и нормализация). Запросы к GraphQL API используются для данных онтологии, а локальное UI-состояние управляется через реактивность Vue (`ref`, `reactive`).

**UI-библиотека:** Pencil.dev — компоненты генерируются из `design/frontend.pen`. Нет сторонней UI-библиотеки; весь UI собирается из дизайн-системы Pencil. Vue компоненты создаются автоматически на основе .pen-файла, что гарантирует pixel-perfect соответствие дизайну.

**UI-дизайн:** Pencil.dev для создания wireframes, прототипов и дизайн-спецификаций.

**Визуализация:** @vue-flow/core (портированный React Flow) с кастомными нодами для классов и свойств.

**E2E-тестирование:** Playwright для end-to-end тестирования фронтенда.

**Линтинг и форматирование:** Biome — единый линтер и форматировщик для TypeScript, TSX и Vue-файлов (замена ESLint + Prettier). Обеспечивает единый стиль кода и статический анализ.

---

### 2. Языки backend

| Язык | Сервисы | Обоснование |
|----------|----------|-----------|
| **Rust** | Ontology Service (ядро), Versioning Service | Максимальная производительность, безопасность памяти, отсутствие GC — критично для обхода графов и Git-операций. Нулевые затраты на абстракции. |
| **Go** | Auth Service | Встроенная конкурентность (goroutines), отличная производительность для I/O-операций, простота развёртывания (статический бинарник). Легко писать gRPC серверы. |
| **Python** | Metrics Service, Notification Service | Быстрая разработка, богатая экосистема для аналитики (pandas, numpy) и работы с очередями (aio-pika). PyO3 для вызова критичных Rust-функций. |

**Рассмотренные альтернативы:**
- Java/Spring Boot — требовала бы больше памяти, сложнее конфигурация, накладные расходы на GC.
- Node.js — не обеспечил бы нужную производительность для работы с графами до 1 млн узлов.

**Инструменты разработки Python:**
- **uv** — единый менеджер пакетов и виртуальных окружений для всех Python-сервисов (замена pip, virtualenv, pip-tools). Используется для управления зависимостями (`uv sync`), сборки и публикации.
- **ruff** — единый линтер и форматировщик для всех Python-сервисов (замена flake8, isort, black). Выполняет статический анализ (`ruff check`) и форматирование (`ruff format`).

---

### 3. Git-подобное версионирование

**Решение:** Реализовать **собственный Versioning Service** на Rust, использующий канонический Turtle (нормализованный) для хранения разницы между коммитами. Git-репозиторий — только экспорт/импорт.

**Рассмотренные альтернативы:**
- **Нативный Git** — проблема нестабильной сериализации RDF/OWL (ложные изменения строк), не умеет работать с ABox на миллионах индивидов, не поддерживает семантическое слияние.
- **Git LFS / LOP** — отложено до появления зрелой поддержки LOP (LOP ещё в разработке).

**Последствия:**
- ✅ Коммиты на уровне семантических изменений (добавление/удаление классов, изменение свойств)
- ✅ Ветвление и слияние с пониманием онтологической структуры
- ✅ Каноническая сериализация Turtle для стабильного diff
- ✅ Экспорт в Git как опция для совместимости с существующими CI/CD
- ❌ Дополнительная реализация (не берём готовый Git)

---

### 4. Наблюдаемость: OpenTelemetry + Grafana Stack

**Решение:** VEDO Core поставляется со встроенным стеком observability на базе OpenTelemetry, Prometheus, Grafana Loki, Grafana Tempo и Grafana. Этот стек является обязательным для базовой поставки и production readiness.

**Компоненты:**

| Компонент | Назначение | Источник данных |
|-----------|---------|-------------|
| **OpenTelemetry Collector** | Сбор и маршрутизация сигналов | OTLP (gRPC/HTTP) |
| **Prometheus** | Хранение метрик (CPU, RAM, latency, error rate) | Metrics |
| **Grafana Tempo** | Хранение трейсов (распределённая трассировка) | Traces |
| **Grafana Loki** | Хранение логов | Logs |
| **Grafana** | Единая дашборд-панель | All |

**Обязательность поставки:**
- **Prometheus** обязателен для сбора и хранения метрик `vedo_*`, RED metrics, latency, error rate и инфраструктурных показателей.
- **Grafana** обязательна как единая панель дашбордов для метрик, логов и трейсов.
- **OpenTelemetry** обязателен как vendor-neutral стандарт сбора трейсов, метрик и логов.
- **Grafana Loki** обязателен для хранения и поиска логов в базовой поставке.
- **Grafana Tempo** обязателен для хранения трейсов в базовой поставке.

**Альтернативы:**
- **Victoria Metrics** может использоваться вместо Prometheus для крупных инсталляций через remote write, но не отменяет поставку Prometheus/Grafana по умолчанию.
- **Jaeger** может использоваться вместо Tempo, если заказчик требует Jaeger-совместимый tracing backend.
- **Datadog** и **New Relic** поддерживаются через OpenTelemetry exporters при наличии лицензии заказчика.
- **Elastic Stack / ELK** допускается как частичная альтернатива для логов и аналитики, но требует адаптации exporters и дашбордов.

**Граница ответственности:** поддержка альтернативных стеков не входит в базовую поставку VEDO Core. Если заказчик выбирает корпоративный стек observability, настройка, лицензии и эксплуатация альтернативного backend являются зоной ответственности заказчика или отдельной платной услугой. VEDO предоставляет шаблоны Grafana dashboards; Kibana dashboards для ELK не входят в поставку по умолчанию.

**Retention по умолчанию:**
- Метрики: 30 дней.
- Логи: 30 дней.
- Трейсы: 7 дней.

Для Enterprise/on-premise инсталляций большого масштаба может предлагаться Victoria Metrics вместо Prometheus как масштабируемый backend метрик.

**Инструментирование:**
- **Rust**: `opentelemetry` + `tracing` + `tracing-opentelemetry`
- **Go**: `otel` + `otelgrpc` + `otelhttp`
- **Python**: `opentelemetry-instrumentation-fastapi`
- **Vue 3**: `@opentelemetry/sdk-trace-web` для фронтенд-трассировки

**Якорные метрики (RED method):**
- **Rate** — количество запросов в секунду
- **Errors** — количество ошибок (по HTTP статусу)
- **Duration** — гистограмма времени ответа (p50, p90, p99)

**Пример трейса для VEDO:**

```yaml
# Трасcировка операции "Создание класса"
- trace_id: "abc123"
  spans:
    - name: "POST /api/v1/ontologies/123/nodes"
      service: "api-gateway"
      duration: 200ms
    - name: "AuthService.VerifyToken"
      service: "auth"
      duration: 5ms
    - name: "OntologyService.CreateNode"
      service: "ontology"
      duration: 150ms
      events:
        - "Validate node"
        - "Check lock"
        - "Write to Neo4j"
```

**Последствия:**
- ✅ Единый стандарт для observability в кластере
- ✅ Бесперебойное отслеживание производительности и узких мест
- ✅ Возможность алертинга при ошибках или высоких задержках
- ❌ Дополнительная инфраструктура (операторы для Grafana, Tempo, Loki) но в рамках одного Helm-чарта управляемо.

---

### 5. Коммуникация и API

| Протокол | Использование | Обоснование |
|----------|-------|-----------|
| **gRPC (Rust/Go)** | Внутренние вызовы между микросервисами | Высокая производительность, контракты через protobuf, потоковая передача |
| **REST (JSON)** | Внешний API для отраслевых приложений | Широкая поддержка, простота, документация через OpenAPI |
| **WebSocket** | Коллаборация в реальном времени (блокировки, комментарии) | Двунаправленная связь, низкая задержка |
| **GraphQL** | Основной API фронтенда для навигации по графу онтологии | Порционная загрузка подграфов, cursor pagination, вложенные запросы, Apollo Client cache; динамические свойства возвращаются через `propertyValues` и `outgoingEdges` |

Граница протоколов обязательна:
- Внешние REST endpoints `/api/v1/...` принадлежат API Gateway.
- API Gateway вызывает внутренние сервисы через gRPC/protobuf, включая Ontology Service.
- Ontology Service и другие внутренние доменные сервисы не публикуют функциональный REST API для CRUD, search, SPARQL, import/export или versioning операций.
- Единственный разрешённый HTTP management surface внутренних сервисов: `GET /`, `GET /health`, `GET /ready`.
- `GET /` каждого сервиса возвращает унифицированный JSON с service metadata; он не содержит доменной бизнес-логики.

---

### 8. Политика выбора языка

**Решение:** Каждый сервис реализуется на языке, указанном в таблице ниже. Это относится как к production-коду, так и к заглушкам (stubs) — заглушка сервиса должна быть написана на том же языке и фреймворке, что и его production-версия.

**Обоснование:**
- Заглушка на целевом языке позволяет рано выявить проблемы инфраструктуры сборки (Cargo.toml, go.mod, package.json, requirements.txt) и CI-пайплайна для каждого языка.
- Разработчик, берущий сервис в реализацию, получает готовую точку входа в знакомом языке и фреймворке.
- Единый язык для заглушки и production исключает расхождение поведения HTTP-хендлеров, middleware и форматов логов при переходе от stub к реализации.

| Сервис | Язык | Фреймворк |
|--------|------|-----------|
| api-gateway | Go | gin |
| auth-service | Go | gin |
| commenting-service | Go | gin |
| ticket-api | Go | gin |
| ticket-telemetry-listener | Go | gin |
| ticket-notifier | Go | gin |
| ontology-service | Rust | actix-web |
| versioning-service | Rust | actix-web |
| publisher-service | Rust | actix-web |
| public-browse-api | Rust | actix-web |
| metrics-service | Python | FastAPI |
| ticket-classifier | Python | FastAPI |
| frontend | TypeScript | Vue 3 + Vite |
| publish-browse-ui | TypeScript | Vue 3 + Vite |

**Исключения:**
- Инфраструктурные сервисы (postgres, neo4j, redis, rabbitmq, minio, keycloak, prometheus, loki, tempo, grafana, otel-collector) остаются официальными Docker-образами без собственной заглушки.
- Сервисы документации (docs-user, docs-dev, docs-admin, docs-integrator) остаются nginx.

**Тип правила:** hard constraint — заглушка обязана быть на том же языке, что и production. Исключения только для сервисов без собственного кода (инфраструктурные).

---

### 6. Базы данных и инфраструктура

| Компонент | Технология | Обоснование |
|-----------|------------|-----------|
| **Triple Store (граф)** | Neo4j (primary) + опционально GraphDB | Neo4j — mature, высокая производительность, Cypher (легче, чем SPARQL). GraphDB — для RDF-нативных сценариев. |
| **Version Store** | PostgreSQL + JSONB | Надёжные транзакции, JSONB для хранения дельт изменений (commits) |
| **Cache & Locks** | Redis Cluster | Скорость, поддержка TTL, простые блокировки |
| **Message Queue** | RabbitMQ (primary) / Kafka (опционально) | Rabbit для надёжной доставки и маршрутизации; Kafka для стримов (event sourcing) |

---

### 7. Документация: Antora

**Решение:** Использовать Antora для сборки проектной и пользовательской документации.

**Роль в стеке:** Antora является documentation site generator для модульной документации VEDO Core: user guides, admin guides, on-premise/air-gap runbooks, API guides, ADR/architecture references и release documentation.

**Обоснование:**
- ✅ Поддерживает docs-as-code: документация хранится в Git рядом с артефактами и кодом.
- ✅ Модульная структура подходит для микросервисов, deployment models и разных аудиторий документации.
- ✅ AsciiDoc лучше Markdown для больших technical docs: includes, attributes, admonitions, cross-references.
- ✅ Подходит для версионированной документации по релизам и LTS/on-premise версиям.
- ✅ Может собираться offline для air-gapped customers как static site без внешних сервисов.

**Рассмотренные альтернативы:**
- **MkDocs** — проще, но Markdown слабее для крупных многостраничных technical docs и сложной cross-reference модели.
- **Docusaurus** — хорош для developer portal, но React/MDX добавляет лишнюю frontend-зависимость и хуже подходит для air-gapped static documentation без JS assumptions.
- **Sphinx** — зрелый инструмент, но Python/RST менее естественны для mixed-language product docs команды.

**Последствия:**
- Документация получает единый build pipeline.
- Нужно стандартизировать AsciiDoc style guide и Antora component/version structure.
- CI должен проверять сборку Antora site и broken links.

---

## Статус

Статус документа — **approved** (утверждён). Все решения зафиксированы для фазы MVP.

## Последствия

- Реализация собственного Versioning Service (вместо нативного Git) потребует дополнительных усилий, но даст семантическое слияние и производительность.
- OpenTelemetry унифицирует observability, но добавляет инфраструктурную сложность (Tempo, Loki, Prometheus).
- Стек (Rust + Go + Python + Vue 3) накладывает требования к CI (разные билды, обратная совместимость gRPC).

## Источники

- [Vue 3 Documentation](https://vuejs.org/)
- [pnpm Documentation](https://pnpm.io/)
- [OpenTelemetry Semantic Conventions](https://opentelemetry.io/docs/specs/semconv/)
- [Grafana O11y Stack](https://grafana.com/oss/)
- Neo4j vs GraphDB comparison (внутренние тесты 2025)
- Rust, Go, Python — внутренние benchmarks команды
