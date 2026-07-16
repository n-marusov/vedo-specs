# ADR-DES.INFRA.ai-orchestration-service-strategy — Выделение AI-оркестрации из API Gateway в отдельный сервис

**Дата:** 2026-07-16
**Статус:** ПРЕДЛОЖЕНО

## Контекст

Проект вводит AI-функции, использующие LLM-провайдеров:

- NL→OWL генерация (F14.1, F14.2) — преобразование естественного языка в OWL-онтологию
- NL-режим запросов (F10.1, F10.2) — запросы к онтологии на естественном языке
- AI-подсказки по связям (F14.3) — рекомендации связей и свойств
- AI-рефайнмент онтологии (F14.5) — улучшение существующей структуры
- Извлечение онтологии из документов (F14.4, F14.6) — через document-extractor

LLM-библиотека (`src/services/shared/llm/`) написана на Go и содержит:
- Провайдеры (OpenAI, Anthropic, локальные OpenAI-совместимые)
- `TemplateEngine` — Go `text/template` для промптов
- `Config`/`Registry`/`Retry`/`Observability`/`Cost` — инфраструктурные абстракции

Текущее размещение AI-хендлеров в API Gateway:

```
src/services/api-gateway/
├── internal/
│   ├── handler/
│   │   ├── nl_to_owl_handler.go      # NL→OWL генерация
│   │   ├── ai_completion_handler.go   # AI-автодополнение
│   │   ├── refinement_handler.go      # Рефайнмент онтологии
│   │   ├── template_handler.go        # Управление промпт-шаблонами
│   │   └── prompt_injection.go        # Prompt Injection фильтр
│   ├── middleware/
│   │   └── llm_policy_router.go       # LLM Policy Router
│   ├── models/
│   │   └── ai.go                      # AI-модели данных
│   └── templates/                     # Промпт-шаблоны (.tmpl)
```

Это решение **противоречит** ключевому архитектурному принципу, зафиксированному в `ARCHITECTURE.md`:

> **«Smart Endpoints, Dumb Pipes:** Business logic lives inside services — API Gateway routes but does not transform or orchestrate.»

AI-хендлеры не являются «маршрутизацией» — они реализуют бизнес-логику:
- Парсинг OWL (nl_to_owl_handler.go)
- Управление состоянием рефайнмента (refinement_handler.go)
- Prompt Injection защита на уровне контента (prompt_injection.go)
- Управление шаблонами промптов (template_handler.go)

Размещение этой логики в Gateway создаёт **Leaky Gateway** — один из задокументированных антипаттернов проекта.

## Требование-источник

**Архитектурные принципы:**
- `.ai-factory/ARCHITECTURE.md` — принцип «Smart Endpoints, Dumb Pipes»; антипаттерн Leaky Gateway
- `ADR-DES.INFRA.monolith-vs-microservices` — микросервисная архитектура, доменная декомпозиция
- `ADR-DES.API.protocol-stack-strategy` — gRPC для внутренних вызовов, Gateway как фасад
- `ADR-IMPL.STACK.api-gateway-go-strategy` — Gateway на Go (ai-orchestration-service наследует язык)

**AI/LLM-специфичные требования:**
- `REQ-FUN.API.llm-policy` — политика маршрутизации LLM-запросов
- `REQ-FUN.DATA.ontology-visibility-levels` — уровни видимости (Public/Internal/Private)
- `REQ-FUN.UI.external-llm-override` — административное разрешение внешних LLM
- `REQ-CON.INFRA.saas-llm-limitations` — ограничения SaaS (нет локальной LLM)
- `REQ-CON.INFRA.on-premise-llm-limitations` — ограничения On-premise (только локальная LLM)
- `REQ-CON.INFRA.on-premise-local-llm` — конфигурация локальной LLM
- `REQ-NFR.INFRA.router-resilience` — отказоустойчивость LLM Router (требования переносятся на ai-orchestration-service)
- `REQ-NFR.SECURITY.prompt-filter-blacklist` — чёрный список промптов
- `REQ-NFR.SECURITY.prompt-structure-detection` — детекция вложенных структур
- `REQ-NFR.SECURITY.system-prompt-hardening` — усиление системного промпта
- `REQ-NFR.SECURITY.prompt-audit` — аудит промптов
- `REQ-NFR.PROCESS.prompt-injection-tests` — тесты промпт-инъекций

**AI-функции (vision.md F14, F10):**
- `vision.md` F14.1 — NL-описание → OWL (генерация онтологии)
- `vision.md` F14.2 — Итеративное уточнение (рефайнмент)
- `vision.md` F14.3 — AI-подсказки по связям (suggestions)
- `vision.md` F10.1, F10.2 — NL-режим запросов (SPARQL/CYPHER)
- `REQ-FUN.API.iterative-refinement-context` — сохранение контекста при уточнении
- `REQ-FUN.API.max-refinement-iterations` — лимит циклов уточнения (≤ 3)
- `REQ-FUN.API.suggestion-types` — типы AI-подсказок
- `REQ-FUN.API.suggestion-confidence-threshold` — порог уверенности подсказок

**Шаблоны и документы:**
- `REQ-FUN.API.templates-catalog` — каталог из 5 шаблонов MVP
- `REQ-FUN.API.templates-definition` — структура шаблона онтологии
- `REQ-FUN.API.templates-import` — импорт шаблонов
- `REQ-FUN.API.templates-versioning` — версионирование шаблонов
- `ADR-DES.INFRA.doc-extractor-service-strategy` — document-extractor (Python, использует LLM)

**Связанные ADR (требуют обновления — см. «Воздействие»):**
- `ADR-DES.API.llm-policy-router-strategy` — LLM Policy Router (размещён в Gateway → мигрирует)
- `ADR-DES.SECURITY.prompt-injection-defense` — защита от промпт-инъекций (уровни 1-3 в Gateway → мигрируют)

## Решение

**Выделить AI-оркестрацию из API Gateway в отдельный микросервис `ai-orchestration-service` (Go) с gRPC-контрактом.**

AI-хендлеры, LLM Policy Router и управление промпт-шаблонами перемещаются из Gateway в новый сервис. API Gateway сохраняет только функцию проксирования запросов к `ai-orchestration-service` (как к любому другому внутреннему сервису).

### Целевая архитектура

```
┌──────────────┐
│   Frontend   │
│   (Vue 3)    │
└──────┬───────┘
       │ HTTP/REST
       ▼
┌─────────────────────────────────────────────────┐
│              API Gateway (Go)                    │
│                                                  │
│  ┌────────────────────────────────────────────┐  │
│  │  REST/gRPC/GraphQL Proxy (чистая            │  │
│  │  маршрутизация, без бизнес-логики)          │  │
│  │                                             │  │
│  │  /api/v1/ai/*  →  gRPC ai-orchestration    │  │
│  │  /api/v1/onto/* → gRPC ontology-service     │  │
│  │  /api/v1/ver/*  → gRPC versioning-service   │  │
│  └────────────────────────────────────────────┘  │
│                                                  │
│  Middleware (без AI-логики):                     │
│  • Auth forwarding (JWT → gRPC metadata)        │
│  • Rate limiting, CORS                          │
│  • Circuit breaker, timeout                     │
│  • OpenTelemetry trace propagation              │
└──────────────────┬──────────────────────────────┘
                   │ gRPC
                   ▼
┌──────────────────────────────────────────────────┐
│         ai-orchestration-service (Go)             │
│                                                  │
│  ┌────────────────┐  ┌────────────────────────┐  │
│  │ LLM Policy     │  │ AI Handlers            │  │
│  │ Router         │  │ • NL→OWL генерация     │  │
│  │ (из Gateway)   │  │ • AI-автодополнение    │  │
│  └────────────────┘  │ • Рефайнмент           │  │
│                      │ • Prompt Injection     │  │
│  ┌────────────────┐  └────────────────────────┘  │
│  │ Template       │                              │
│  │ Engine         │  ┌────────────────────────┐  │
│  │ (shared/llm)   │  │ gRPC Server            │  │
│  └────────────────┘  │ (proto-контракт)       │  │
│                      └───────────┬────────────┘  │
│  ┌────────────────┐              │                │
│  │ LLM Adapters   │              │                │
│  │ • OpenAI       │              │                │
│  │ • Anthropic    │              │                │
│  │ • Local        │              │                │
│  └────────────────┘              │                │
└──────────────────────────────────┼────────────────┘
                                   │ gRPC
                    ┌──────────────┼──────────────┐
                    ▼              ▼              ▼
             ┌──────────┐  ┌──────────┐  ┌──────────────┐
             │ontology  │  │versioning│  │document      │
             │-service  │  │-service  │  │-extractor    │
             │(Rust)    │  │(Rust)    │  │(Python)      │
             └──────────┘  └──────────┘  └──────────────┘
```

### gRPC-контракт ai-orchestration-service

```protobuf
// proto/ai-orchestration/v1/ai_orchestration.proto

service AIOrchestrationService {
  // NL → OWL генерация
  rpc GenerateOWL(GenerateOWLRequest) returns (GenerateOWLResponse);

  // NL-запрос к онтологии
  rpc NaturalLanguageQuery(NLQueryRequest) returns (NLQueryResponse);

  // AI-рефайнмент онтологии
  rpc RefineOntology(RefineOntologyRequest) returns (stream RefineOntologyEvent);

  // AI-автодополнение (стриминг для UX)
  rpc Complete(CompletionRequest) returns (stream CompletionChunk);

  // Управление промпт-шаблонами
  rpc ListTemplates(ListTemplatesRequest) returns (ListTemplatesResponse);
  rpc GetTemplate(GetTemplateRequest) returns (GetTemplateResponse);
  rpc UpdateTemplate(UpdateTemplateRequest) returns (UpdateTemplateResponse);

  // Policy and audit for external services
  rpc CheckPolicy(CheckPolicyRequest) returns (CheckPolicyResponse);
  rpc LogLLMUsage(LogLLMUsageRequest) returns (LogLLMUsageResponse);
}

message GenerateOWLRequest {
  string ontology_id = 1;
  string natural_language_input = 2;
  string target_language = 3;       // OWL Functional, Manchester, Turtle
  optional string system_prompt_override = 4;
}

message GenerateOWLResponse {
  string owl_content = 1;
  string target_language = 2;
  repeated AxiomWarning warnings = 3;
  TokenUsage token_usage = 4;
}

message CheckPolicyRequest {
  string ontology_id = 1;
  string action = 2;  // document_extraction, nl_query, suggestion
}

message CheckPolicyResponse {
  bool allowed = 1;
  string provider = 2;   // openai, anthropic, local
  string model = 3;
  string reason = 4;     // if blocked: why
  bool require_consent = 5;
}

message LogLLMUsageRequest {
  string ontology_id = 1;
  string action = 2;
  string provider = 3;
  string model = 4;
  int32 tokens_in = 5;
  int32 tokens_out = 6;
  double cost = 7;
  string trace_id = 8;
}

message LogLLMUsageResponse {}
```

### Что переезжает из Gateway в ai-orchestration-service

| Компонент Gateway | Назначение в ai-orchestration-service |
|---|---|
| `nl_to_owl_handler.go` | gRPC-хендлер `GenerateOWL` |
| `ai_completion_handler.go` | gRPC-хендлер `Complete` (server-streaming) |
| `refinement_handler.go` | gRPC-хендлер `RefineOntology` (bidirectional streaming) |
| `template_handler.go` | gRPC-хендлеры `ListTemplates`/`GetTemplate`/`UpdateTemplate` |
| `prompt_injection.go` | Middleware в ai-orchestration-service (перед вызовом LLM) |
| `llm_policy_router.go` | Middleware в ai-orchestration-service (перед вызовом LLM) |
| `models/ai.go` | Переиспользуется как общие модели через `shared/llm` |
| `internal/templates/` | Переезжают в ai-orchestration-service |

### Что остаётся в API Gateway

- Только gRPC-прокси для `/api/v1/ai/*` → `ai-orchestration-service`
- Аутентификация (JWT-валидация, проброс claims в gRPC metadata)
- Rate limiting, CORS, circuit breaker — как для любого другого сервиса
- **Никакой AI-бизнес-логики**


### Интеграция других сервисов с ai-orchestration-service

После выделения ai-orchestration-service другие сервисы, использующие LLM, интегрируются с ним следующим образом:

#### document-extractor (Python)

document-extractor сохраняет собственный LLM-клиент (httpx-based) и промпты, но получает политики и аудит через gRPC:

```
doc-extractor получает запрос
  |
  +-- gRPC -> ai-orchestration.CheckPolicy(ontology_id, "document_extraction")
  |          <- { allowed: true, provider: "openai", model: "gpt-4o-mini" }
  |
  +-- httpx -> LLM Provider (используя provider/model из ответа)
  |
  +-- gRPC -> ai-orchestration.LogLLMUsage(tokens_in, tokens_out, cost, ...)
  |
  +-- gRPC -> ontology-service.ApplySequence(...)
```

**Обоснование (Hybrid Model):**
- **Pass-through не подходит:** doc-extractor использует специализированные Python-промпты, перенос которых в Go-сервис неоправдан
- **Policy-only недостаточно:** нужен централизованный аудит LLM-вызовов
- **Выбрана Hybrid Model:** CheckPolicy перед вызовом + LogLLMUsage после. doc-extractor сохраняет полный контроль над LLM-промптами и логикой

#### MCP-сервер (greenfield)

MCP-сервер проектируется как **тонкий прокси** без собственного LLM-клиента:

```
MCP Client (Claude Desktop, Cursor)
       | MCP protocol (JSON-RPC)
       v
+------------------+
|   MCP Server     |  Go, gRPC clients only
|                  |
|  + MCP handler   |
|  + gRPC -> ai-orchestration.NaturalLanguageQuery()
|  + gRPC -> ontology-service.ExecuteQuery()
+------------------+
```

**Обоснование (Thin Proxy Model):**
- NL->Query трансляция — та же, что F10.1/F10.2 — дублирование недопустимо
- LLM-политики и prompt injection defense — централизованы в ai-orchestration-service
- MCP-сервер остаётся лёгким и не содержит AI-бизнес-логики
- Полное соответствие принципу Smart Endpoints, Dumb Pipes


## Рассмотренные альтернативы

| Альтернатива | Описание | Причина отклонения |
|--------------|----------|--------------------|
| **A: Оставить в Gateway навсегда** | AI-хендлеры остаются частью API Gateway | Нарушает «Smart Endpoints, Dumb Pipes». Создаёт Leaky Gateway антипаттерн. Затрудняет независимое масштабирование AI-операций (CPU/GPU-intensive) и routing (I/O-bound). AI-изменения требуют редеплоя Gateway — риск для всей платформы. |
| **B: Выделить ai-orchestration-service (выбрано)** | Отдельный Go-микросервис с gRPC | **Принято.** Соответствует архитектурным принципам. Независимое масштабирование и деплой. |
| **C: По одному сервису на AI-функцию** | Отдельные сервисы: `nl-to-owl-service`, `refinement-service`, `completion-service` | Nano-services антипаттерн — operational overhead превышает бизнес-ценность. Все AI-функции разделяют LLM-библиотеку, LLM Policy Router и Template Engine — естественная cohesion в одном сервисе. |
| **D: AI-логика в ontology-service (Rust)** | Расширение ontology-service AI-функциями | LLM-библиотека на Go, а не на Rust. Потребовало бы полного переписывания `shared/llm` на Rust. Нарушает принцип единой ответственности — ontology-service должен управлять графом, а не LLM-оркестрацией. |

## Последствия

### Положительные последствия

- **Архитектурная чистота:** Соответствие принципу «Smart Endpoints, Dumb Pipes». API Gateway — чистый прокси без бизнес-логики.
- **Независимое масштабирование:** AI-операции (CPU/GPU-intensive вызовы LLM) масштабируются независимо от routing (I/O-bound). При росте AI-нагрузки не затрагивается стабильность Gateway.
- **Изолированные деплои:** Изменения AI-логики не требуют редеплоя Gateway. AI-эксперименты и A/B-тесты промптов не рискуют стабильностью платформы.
- **Чёткий API-контракт:** gRPC proto-файл — self-documenting контракт, по которому frontend (через Gateway) и другие сервисы взаимодействуют с AI-функциями.
- **Cohesion:** LLM Policy Router, Prompt Injection защита, Template Engine и AI-хендлеры естественно объединены в одном сервисе — все они работают с LLM-взаимодействиями.
- **Упрощение Gateway:** Gateway содержит только middleware общего назначения (auth, rate limiting, tracing), без домен-специфичной логики.
- **Тестируемость:** AI-сервис можно тестировать изолированно, с mock-LLM-провайдерами, без поднятия всего Gateway.
- **Переиспользование `shared/llm`:** Библиотека уже на Go, миграция тривиальна — импорт вместо копирования.

### Отрицательные последствия

**Новый сервис — новый operational overhead:** Дополнительный Docker-контейнер, health-чеки, мониторинг, CI/CD-пайплайн, деплой.
- **Дополнительный сетевой hop:** Каждый AI-запрос проходит Frontend → Gateway → ai-orchestration → LLM Provider, добавляя ~1-2 мс gRPC latency по сравнению с прямым вызовом из Gateway.
- **Синхронизация proto-контрактов:** Изменения API требуют регенерации кода на стороне Gateway (Go) и при необходимости — Frontend (через grpc-gateway для REST).
- **Миграция состояния:** Если refinement_handler.go использует in-memory состояние в Gateway, его нужно перенести в ai-orchestration-service (или вынести в Redis).
- **document-extractor (Python) не может использовать Go-библиотеку `shared/llm` напрямую:** После миграции doc-extractor использует собственный LLM-клиент для вызовов LLM и обращается к ai-orchestration-service через gRPC для enforcement LLM-политик вместо дублирования логики политик (см. секцию «Решённые вопросы»).

### Меры снижения рисков

- **Мониторинг латентности:** OpenTelemetry span `ai_orchestration.*` с алертом при p95 > 200 мс (включая время LLM).
- **Circuit breaker:** Gateway настраивает circuit breaker для ai-orchestration-service — при деградации сервиса AI-функции gracefully unavailable, остальная платформа работает.
- **Proto-версионирование:** Пакет `ai-orchestration/v1` позволяет эволюцию API без разрыва обратной совместимости.
- **State migration:** Состояние рефайнмента выносится в Redis с TTL — stateless сервис, горизонтальное масштабирование.
- **Feature flag:** `ai.orchestration.service.enabled` — позволяет переключаться между Gateway-хендлерами и gRPC-сервисом без даунтайма.
- **Интеграционные тесты:** Полный набор тестов для gRPC-контракта до миграции, чтобы гарантировать идентичное поведение.

## Воздействие на существующие артефакты

Выделение ai-orchestration-service затрагивает следующие артефакты в `specs/`. Ниже перечислены требуемые изменения для каждого.

### Требуют обновления (конфликт реализации)

| Артефакт | Проблема | Требуемое изменение 
|----------|----------|---------------------|------|
| `REQ-FUN.API.llm-policy.md:41` | «Политика применяется на уровне API Gateway» | Заменить на «Политика применяется на уровне ai-orchestration-service до вызова LLM» или убрать упоминание конкретного сервиса |
| `ADR-DES.API.llm-policy-router-strategy` (арх. схема) | LLM Policy Router изображён внутри блока «API Gateway» | Добавить addendum: LLM Policy Router мигрирует в ai-orchestration-service. Существующая схема остаётся валидной для промежуточного состояния. |
| `ADR-DES.SECURITY.prompt-injection-defense` (арх. схема) | Уровни 1-3 изображены внутри блока «API Gateway» | Добавить addendum: уровни 1-3 становятся middleware ai-orchestration-service. Существующая схема остаётся валидной для промежуточного состояния. |
| `REQ-NFR.INFRA.router-resilience.md:17` | «компонента LLM Router (API Gateway)» — привязка к Gateway | Заменить на «компонента LLM Router (ai-orchestration-service)». Требования к отказоустойчивости без изменений. |

### Требуют дополнения (новый сервис)

| Артефакт | Требуемое изменение 
|----------|---------------------|------|
| `.ai-factory/ARCHITECTURE.md` | Добавить `ai-orchestration-service` в перечень сервисов и диаграмму зависимостей |
| `deploy/docker-compose.yml` | Добавить секцию `ai-orchestration-service` |
| `deploy/helm/` | Добавить Helm-chart для ai-orchestration-service |
| `deploy/observability/` | Добавить дашборды Grafana, алерты Prometheus |

### Решённые вопросы (результаты исследования 2026-07-16)

| Вопрос | Решение | Модель | Детали |
|--------|---------|--------|--------|
| **document-extractor и LLM-политики** | doc-extractor вызывает CheckPolicy перед LLM-вызовом и LogLLMUsage после | **Hybrid Model** | Собственный LLM-клиент сохраняется. Политики — через gRPC. Промпты остаются в Python. См. секцию «Интеграция других сервисов». |
| **MCP-сервер и ai-orchestration** | MCP-сервер — тонкий gRPC-прокси, не содержит LLM-клиента | **Thin Proxy Model** | NL->Query через NaturalLanguageQuery(). Query execution через ontology-service. См. секцию «Интеграция других сервисов». |
| **Формат аудита LLM-вызовов** | LogLLMUsage — синхронный gRPC | gRPC unary call | При высоких объёмах (>100 rps) рассмотреть асинхронную очередь (RabbitMQ) в отдельном ADR |

## Связанные ADR

| ADR | Связь | Действие |
|-----|-------|----------|
| `ADR-DES.API.llm-policy-router-strategy` | LLM Policy Router мигрирует из Gateway в ai-orchestration-service | Требуется обновление архитектурной схемы |
| `ADR-DES.SECURITY.prompt-injection-defense` | Уровни защиты 1-3 становятся middleware ai-orchestration-service | Требуется обновление архитектурной схемы |
| `ADR-DES.INFRA.doc-extractor-service-strategy` | document-extractor использует CheckPolicy+LogLLMUsage через gRPC (Hybrid Model) | Требуется обновление: добавить gRPC-вызовы CheckPolicy/LogLLMUsage в doc-extractor |
| `ADR-DES.INTEGRATION.mcp-server-query-adoption` | MCP-сервер — тонкий прокси: NL->Query через ai-orchestration (Thin Proxy Model) | Требуется обновление: пересмотреть архитектуру MCP-сервера (убрать LLM-клиент) |
| `ADR-DES.INFRA.monolith-vs-microservices` | Обоснование микросервисной декомпозиции | Без изменений |
| `ADR-DES.API.protocol-stack-strategy` | gRPC для внутренних вызовов | Без изменений |
| `ADR-IMPL.STACK.api-gateway-go-strategy` | ai-orchestration-service наследует Go | Без изменений |
| `ADR-IMPL.STACK.microservice-language-stack-strategy` | ai-orchestration-service — I/O-bound, категория Go | Без изменений |
| `ADR-DES.API.sparql-query-language-strategy` | NL-режим запросов через ai-orchestration-service | Без изменений (маршрут обновляется) |
| `ADR-DES.SECURITY.nl-query-opt-in-mandate` | Opt-in диалог — enforcement через ai-orchestration-service | Без изменений (логика та же) |
| `ADR-DES.INFRA.airgap-offline-deployment-strategy` | On-premise поставка включает ai-orchestration-service | Без изменений (новый компонент) |

## Чек-лист реализации

- [ ] Определить proto-контракт `ai-orchestration/v1/ai_orchestration.proto`
- [ ] Создать Go-сервис `src/services/ai-orchestration-service/` со структурой:
  - `main.go` — gRPC-сервер
  - `internal/handler/` — gRPC-хендлеры (мигрированные из Gateway)
  - `internal/middleware/` — LLM Policy Router, Prompt Injection
  - `internal/templates/` — промпт-шаблоны
- [ ] Перенести `shared/llm` как зависимость (go.mod)
- [ ] Мигрировать AI-хендлеры из Gateway в ai-orchestration-service
- [ ] Добавить gRPC-прокси в Gateway для `/api/v1/ai/*` → `ai-orchestration-service`
- [ ] Реализовать circuit breaker для ai-orchestration-service в Gateway
- [ ] Добавить сервис в `docker-compose.yml`
- [ ] **doc-extractor:** добавить gRPC-вызовы CheckPolicy и LogLLMUsage
- [ ] **doc-extractor:** обновить ADR-DES.INFRA.doc-extractor-service-strategy
- [ ] **MCP-сервер:** спроектировать как тонкий прокси (без LLM-клиента)
- [ ] **MCP-сервер:** обновить ADR-DES.INTEGRATION.mcp-server-query-adoption
- [ ] Обновить `ARCHITECTURE.md`: добавить ai-orchestration-service в перечень сервисов
- [ ] Обновить `REQ-FUN.API.llm-policy.md`: убрать привязку к API Gateway
- [ ] Обновить `REQ-NFR.INFRA.router-resilience.md`: заменить (API Gateway) на (ai-orchestration-service)
- [ ] Добавить addendum о миграции в `ADR-DES.API.llm-policy-router-strategy`
- [ ] Добавить addendum о миграции в `ADR-DES.SECURITY.prompt-injection-defense`

---
