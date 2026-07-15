# ADR-DES.DATA.mcp-audit-strategy — Стратегия аудита MCP-сервера

**Статус:** ПРЕДЛОЖЕНО
**Дата:** 2026-07-15

## Контекст

VEDO Hub предоставляет MCP-сервер (F10.3) для внешних LLM-клиентов (Claude Desktop, Cursor, Continue, Zed и др.). MCP-сервер публикует три инструмента: `query-sparql`, `query-cypher`, `introspect-ontology`.

MCP-сервер принят ADR `ADR-DES.INTEGRATION.mcp-server-query-adoption`, в котором зафиксировано требование: «Покрыть инструменты теми же ограничениями безопасности (read-only, LIMIT, timeout, rate limiting, query complexity, **аудит**)». Однако детальная стратегия аудита MCP не была разработана.

Без аудита невозможно:
- Расследовать инциденты, вызванные через MCP-инструменты
- Восстановить цепочку доказательств (chain of custody)
- Соответствовать compliance-требованиям (GDPR/152-ФЗ)
- Отслеживать аномальную активность внешних клиентов

**Существующая инфраструктура (C4 container + deployment):**
- **Support DB (PostgreSQL):** support metadata, tenant info, break-glass audit (уже существует)
- **S3/MinIO (Хранилище реплик):** WAL, snapshots, off-site copy (уже существует)
- **Vault/KMS:** key management, encryption keys (уже существует)
- **Grafana Stack:** Prometheus + Loki + Tempo для observability (уже существует)
- **OpenTelemetry:** trace propagation через все сервисы (уже существует)

## Требование-источник

- `REQ-FUN.INTEGRATION.audit-log-content` — состав полей audit-записи
- `REQ-NFR.DATA.audit-retention` — сроки хранения (90/365 дней)
- `REQ-FUN.INTEGRATION.full-response-storage` — хранение полных ответов в S3
- `REQ-USR.UI.audit-access` — матрица доступа к логам
- `REQ-USR.UI.audit-ui` — интерфейс просмотра логов
- `REQ-NFR.SECURITY.audit-masking` — маскировка sensitive data
- `REQ-FUN.INTEGRATION.audit-api` — API для доступа к логам
- `REQ-FUN.INTEGRATION.audit-integration` — интеграция с общей системой аудита
- `REQ-NFR.SECURITY.audit-access-audit` — аудит доступа к audit-логам
- `REQ-USR.UI.audit-export-incident` — экспорт для расследования инцидентов
- `REQ-NFR.SECURITY.audit-encryption` — шифрование и защита

## Решение

Интегрировать аудит MCP в **существующую инфраструктуру VEDO Core** без создания новых компонентов:

### 1. Хранение — использование существующих хранилищ

**Audit-логи (метаданные):** → **Support DB** (PostgreSQL, уже существует)
- Support DB уже хранит «support metadata, tenant info, break-glass audit» (C4 container.md, строка 37)
- MCP audit-логи естественно вписываются в категорию «support metadata» и «break-glass audit»
- Добавляется одна таблица `mcp_audit_logs` с индексами

**Полные ответы:** → **S3/MinIO (Хранилище реплик)**, существующее хранилище
- Хранилище реплик уже используется для WAL, snapshots (C4 deployment.md, строка 35)
- Полные MCP-ответы сохраняются с префиксом `mcp-audit/` и TTL 90 дней через S3 lifecycle policy
- Разделение: метаданные в PostgreSQL (быстрый поиск), полные ответы в S3 (дешёвое хранение)

**Шифрование:** → **Vault/KMS** (уже существует)
- Ключи шифрования для S3-объектов управляются через существующий Vault/KMS (C4 container.md, строка 44)

### 2. Observability — использование существующего стека

**Трассировка:** → **OpenTelemetry** (уже внедрён во все сервисы)
- Каждый MCP-вызов создаёт span в существующем trace-контексте
- `trace_id` связывает MCP audit-запись с Tempo traces и Loki-логами

**Метрики:** → **Prometheus** (часть Grafana Stack)
- Дополнительные метрики: `mcp_audit_calls_total`, `mcp_audit_errors_total`
- Существующий дашборд дополняется панелью «MCP Audit Health»

**Алерты:** → **существующая система алертов** (Prometheus → Alertmanager → PagerDuty/Slack)
- Аномальный доступ к audit-логам (> 10 запросов за 5 минут) → P2

### 3. Интеграция — без создания новых сервисов

Аудит MCP реализуется как **middleware в API Gateway** (существующий компонент) без создания отдельного сервиса:

```
┌─────────────────────────────────────────────────────────────────────────┐
│                          API Gateway (существующий)                    │
│  ┌───────────────────────────────────────────────────────────────────┐ │
│  │               MCP Audit Middleware (новый middleware)            │ │
│  │  1. Mask sensitive data                                          │ │
│  │  2. Execute MCP tool                                             │ │
│  │  3. Store audit record → Support DB (PostgreSQL)                 │ │
│  │  4. Store full response → S3/MinIO (Хранилище реплик)           │ │
│  │  5. Create OpenTelemetry span                                    │ │
│  └───────────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────────────┘
                │                       │                   │
                ▼                       ▼                   ▼
┌─────────────────────┐ ┌─────────────────────┐ ┌─────────────────────┐
│  Support DB         │ │  S3/MinIO           │ │  Grafana Stack      │
│  (существующий)     │ │  (существующий)     │ │  (существующий)     │
│                     │ │                     │ │                     │
│  mcp_audit_logs     │ │  mcp-audit/{id}.json│ │  Prometheus метрики │
│  (новая таблица)    │ │  (новый префикс)    │ │  Tempo traces       │
└─────────────────────┘ └─────────────────────┘ └─────────────────────┘
```

**Архитектурное обоснование использования существующей инфраструктуры:**

| Существующий компонент | Использование для MCP-аудита | Преимущество |
|------------------------|------------------------------|--------------|
| **Support DB** | Хранение метаданных audit-логов | Уже развёрнут, уже для «break-glass audit», не требует нового кластера |
| **S3/MinIO (Хранилище реплик)** | Хранение полных ответов | Уже используется для WAL/snapshots, дешёвое хранение с TTL |
| **Vault/KMS** | Управление ключами шифрования | Единая точка управления секретами |
| **Grafana Stack** | Метрики, traces, алерты | Единый observability-стек |
| **OpenTelemetry** | trace_id для корреляции | Уже внедрён во все сервисы |

---

### 4. Структура данных

**Support DB — новая таблица `mcp_audit_logs`:**

```sql
CREATE TABLE mcp_audit_logs (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    timestamp TIMESTAMPTZ NOT NULL DEFAULT now(),
    user_id UUID,
    tenant_id UUID NOT NULL REFERENCES support_tenants(id),
    tool_name VARCHAR(100) NOT NULL CHECK (tool_name IN ('query-sparql', 'query-cypher', 'introspect-ontology')),
    parameters JSONB,
    request_preview TEXT NOT NULL,  -- полный NL-запрос
    response_truncated VARCHAR(500) NOT NULL,  -- truncated до 500 символов
    response_full_url VARCHAR(500),  -- ссылка на S3: mcp-audit/{tenant_id}/{yyyy}/{mm}/{dd}/{id}.json
    response_full_size INTEGER,
    status VARCHAR(20) NOT NULL CHECK (status IN ('success', 'error')),
    duration_ms INTEGER NOT NULL,
    error_message TEXT,
    trace_id VARCHAR(255),
    source VARCHAR(50) NOT NULL,  -- claude_desktop, cursor, continue, zed, direct_api
    ip_address INET
);

-- Индексы для поиска и retention
CREATE INDEX idx_mcp_audit_tenant_time ON mcp_audit_logs(tenant_id, timestamp DESC);
CREATE INDEX idx_mcp_audit_user ON mcp_audit_logs(user_id);
CREATE INDEX idx_mcp_audit_tool ON mcp_audit_logs(tool_name);
CREATE INDEX idx_mcp_audit_trace ON mcp_audit_logs(trace_id);
CREATE INDEX idx_mcp_audit_status ON mcp_audit_logs(status);

-- Partition по месяцам для эффективного удаления
CREATE TABLE mcp_audit_logs (
    ...
) PARTITION BY RANGE (timestamp);

-- Автоматическое создание partitions (cron/pg_partman)
```

**S3/MinIO — полные ответы:**

```
Путь: s3://vedo-replica-store/mcp-audit/{tenant_id}/{yyyy}/{mm}/{dd}/{id}.json

S3 Lifecycle Policy:
- Expiration: 90 days (автоматическое удаление)
```

---

### 5. Матрица доступа (использование существующей RBAC)

| Роль | Доступ | Инфраструктура |
|------|--------|---------------|
| **Пользователь** | Только свои логи | API Gateway → Support DB (фильтр по user_id) |
| **Администратор tenant** | Все логи tenant + полные ответы | API Gateway → Support DB + S3 presigned URL |
| **Поддержка VEDO** | Ограниченные метаданные | API Gateway → Support DB (masked) |
| **Администратор VEDO** | Полный доступ (только при инцидентах) | С аудитом каждого обращения |

---

### 6. Retention (автоматическое удаление)

| Тип данных | Срок | Механизм |
|------------|------|----------|
| **Полные ответы (S3)** | 90 дней | S3 Lifecycle Policy |
| **Метаданные (Support DB)** | 365 дней | pg_partman — автоматическое detach/drop старых partitions |
| **Агрегированная аналитика** | Бессрочно | Обезличенные данные, материализованные представления |

---

### 7. Маскировка sensitive data (до записи)

Маскировка выполняется middleware в API Gateway перед сохранением:

```yaml
masking_patterns:
  - pattern: '(api_key|apikey)["\']?\s*[:=]\s*["\'][^"\']+'
    replacement: '\1="[REDACTED]"'
  - pattern: '(password|passwd)["\']?\s*[:=]\s*["\'][^"\']+'
    replacement: '\1="[REDACTED]"'
  - pattern: 'Authorization:\s*Bearer\s+\S+'
    replacement: 'Authorization: Bearer [REDACTED]'
```

---

### 8. Аудит доступа к audit-логам (chain of custody)

Каждое обращение к `mcp_audit_logs` логируется в отдельную таблицу `mcp_audit_access_log`:

```sql
CREATE TABLE mcp_audit_access_log (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    timestamp TIMESTAMPTZ NOT NULL DEFAULT now(),
    user_id UUID NOT NULL,
    action VARCHAR(50) NOT NULL,  -- view_list, view_detail, export, access_full_response
    resource_id UUID,  -- id просмотренной audit-записи
    ip_address INET,
    trace_id VARCHAR(255)
);
CREATE INDEX idx_mcp_access_user_time ON mcp_audit_access_log(user_id, timestamp DESC);
```

Алерт: `rate(mcp_audit_access_total[5m]) > 10` → P2 «аномальная активность доступа к audit-логам».

---

## Рассмотренные альтернативы

| Альтернатива | Причина отклонения |
|--------------|--------------------|
| Отдельный PostgreSQL-кластер для MCP-аудита | Support DB уже существует и предназначен для «break-glass audit»; новый кластер — избыточная сложность |
| Новый S3-бакет для полных ответов | Хранилище реплик уже существует; новый бакет — дополнительная конфигурация без выгоды |
| Отдельный сервис аудита | Middleware в API Gateway достаточно; новый сервис — усложнение архитектуры без необходимости |
| Только метаданные без полных результатов | Невозможность расследования инцидентов без данных о том, что было возвращено |
| Хранение полных результатов в Support DB | Ограничения по размеру; S3 дешевле и лучше подходит для больших бинарных объектов |
| Без маскировки sensitive data | Риск утечки при доступе поддержки к логам |

## Последствия

**Положительные последствия:**

1. **Минимальное воздействие на архитектуру:** Используется существующая инфраструктура — Support DB, S3/MinIO, Vault/KMS, Grafana Stack. Новых сервисов не создаётся.
2. **Единая observability:** Trace_id связывает MCP-вызов с Tempo traces и Loki-логами через существующий OpenTelemetry.
3. **Экономия ресурсов:** Не требуется разворачивать новые кластеры БД или хранилища.
4. **Консистентность с существующим аудитом:** Support DB уже используется для «break-glass audit».
5. **Автоматический retention:** S3 Lifecycle Policy + pg_partman — без ручного вмешательства.

**Отрицательные последствия:**

1. **Нагрузка на Support DB:** Дополнительные записи в существующий кластер.
2. **Объём S3:** Полные ответы занимают место (хотя и с TTL 90 дней).
3. **Связность:** Зависимость от Support DB и S3 для аудита.

**Меры снижения рисков:**

1. **Нагрузка на Support DB:** Мониторинг через Prometheus; при необходимости — вертикальное масштабирование.
2. **Объём S3:** Сжатие (gzip), TTL 90 дней, мониторинг размера.
3. **Связность:** Асинхронная запись в S3 (после ответа клиенту); graceful degradation при недоступности S3 (логирование без полного ответа).

**Риски:**

1. **Потеря логов при сбое Support DB:** Логирование через стандартный механизм репликации PostgreSQL (primary + 2 standby).
2. **Несанкционированный доступ администратора VEDO:** Аудит каждого обращения, ограничение доступа только при активных инцидентах.
3. **Рост объёма логов:** Автоматический retention через pg_partman и S3 Lifecycle.

## Связанные ADR

- `ADR-DES.INTEGRATION.mcp-server-query-adoption` — MCP-сервер (источник требования аудита)
- `ADR-DES.INFRA.otel-observability-strategy` — OpenTelemetry стек (trace_id, метрики)
- `ADR-DES.INFRA.telemetry-retention-strategy` — Хранение телеметрии (retention-политики)
- `ADR-DES.INFRA.vedo-cli-diagnostics-entrypoint` — CLI диагностика (использование trace_id)
- `ADR-DES.SECURITY.authorization-policy-gates-strategy` — Гейты авторизации (RBAC для доступа)

## Чек-лист реализации

- [ ] Создание таблицы `mcp_audit_logs` в Support DB (с partitioning по месяцам)
- [ ] Создание таблицы `mcp_audit_access_log` в Support DB (аудит доступа)
- [ ] Реализация MCP Audit Middleware в API Gateway
- [ ] Интеграция с S3/MinIO (сохранение полных ответов, lifecycle policy)
- [ ] Маскировка sensitive data в middleware
- [ ] Интеграция с OpenTelemetry (создание spans, trace_id)
- [ ] Prometheus-метрики и Grafana-дашборд
- [ ] Алерты (аномальный доступ, ошибки)
- [ ] UI для просмотра логов (административная панель)
- [ ] API `/api/v1/audit/mcp` (GET, экспорт)
- [ ] Retention: pg_partman для автоматического удаления старых partitions
- [ ] S3 Lifecycle Policy (90 дней)
- [ ] Документация Admin Guide: аудит MCP-сервера

---
