# ADR-DES.DATA.query-export-strategy — Стратегия экспорта сохранённых запросов пользователя

**Статус:** ПРЕДЛОЖЕНО
**Дата:** 2026-07-16

## Контекст

VEDO Hub позволяет пользователям сохранять SPARQL и CYPHER запросы для повторного использования (F10 — Гибкие запросы к графу знаний, библиотека сохранённых запросов). При удалении аккаунта пользователь теряет все сохранённые запросы.

**Проблемы:**

1. **Потеря данных:** Пользователь теряет все сохранённые запросы без возможности их сохранить (Google Reader shutdown, 2013)
2. **GDPR Article 20 (Right to data portability):** Пользователь имеет право на получение своих данных в структурированном, общеупотребительном и машиночитаемом формате
3. **Отсутствие механизма:** Нет способа экспортировать сохранённые запросы

**Требуется** стратегия экспорта, которая:
1. Экспортирует все SPARQL/CYPHER запросы с метаданными
2. Использует удобный формат (ZIP-архив)
3. Соответствует GDPR Article 20
4. Доступна в любой момент (не только при удалении)
5. Безопасна (временные ссылки, шифрование, аудит)

**Существующая инфраструктура (C4 container + deployment):**
- **PostgreSQL кластер** (Version Store) — для метаданных экспортных заданий
- **S3/MinIO (Хранилище реплик)** — для ZIP-архивов (уже используется для WAL/snapshots)
- **SMTP-шлюз** — для email-уведомлений (уже используется Ticket Notifier)
- **OpenTelemetry** — для trace_id и мониторинга

**Согласование с существующими политиками:**
- 30-дневный срок экспорта при удалении = 30-дневный период охлаждения из `REQ-NFR.DATA.account-closure-retention`
- 7-дневный срок ссылки для обычного экспорта = 7-дневная ссылка из `REQ-NFR.DATA.decommission-export`

## Требование-источник

- `REQ-FUN.DATA.query-export-format`
- `REQ-FUN.DATA.gdpr-export-mandate`
- `REQ-FUN.DATA.query-export-content`
- `REQ-USR.UI.query-export-availability`
- `REQ-NFR.DATA.query-export-retention`
- `REQ-FUN.API.query-export-api`
- `REQ-USR.UI.query-export-notifications`
- `REQ-FUN.DATA.query-export-validation`
- `REQ-NFR.DATA.query-export-auto-delete`
- `REQ-NFR.DATA.query-export-scaling`
- `REQ-USR.UI.query-export-localization`
- `REQ-NFR.DATA.query-export-audit`

## Решение

Внедрить **Query Export Service** — лёгкий компонент в составе API Gateway для экспорта сохранённых запросов.

### 1. Архитектурная схема

```
┌─────────────────────────────────────────────────────────────────────────┐
│                          API Gateway (существующий)                    │
│  ┌───────────────────────────────────────────────────────────────────┐ │
│  │  Query Export:                                                    │ │
│  │  1. Сбор запросов пользователя (SPARQL + CYPHER)                 │ │
│  │  2. Генерация manifest.json + SHA-256 hash                       │ │
│  │  3. Создание ZIP (.sparql/.cypher + manifest.json)               │ │
│  │  4. Загрузка в S3/MinIO, запись в PostgreSQL                     │ │
│  │  5. Уведомление через SMTP-шлюз (если асинхронно)               │ │
│  └───────────────────────────────────────────────────────────────────┘ │
│                                                                        │
│  API Endpoints (новые):                                                │
│  POST /api/v1/user/queries/export             — запуск экспорта       │
│  GET  /api/v1/user/queries/export/{job_id}    — статус               │
│  GET  /api/v1/user/queries/export/{job_id}/download — скачивание     │
└─────────────────────────────────────────────────────────────────────────┘
                │                   │                   │
                ▼                   ▼                   ▼
┌─────────────────────┐ ┌─────────────────────┐ ┌─────────────────────┐
│  PostgreSQL         │ │  S3/MinIO           │ │  SMTP-шлюз          │
│  (существующий)     │ │  (существующий)     │ │  (существующий)     │
│                     │ │                     │ │                     │
│  query_exports      │ │  exports/queries/   │ │  Email: готовность, │
│  (новая таблица)    │ │  (новый префикс)    │ │  напоминание        │
└─────────────────────┘ └─────────────────────┘ └─────────────────────┘
```

### 2. Структура данных

**PostgreSQL:**

```sql
CREATE TABLE query_exports (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL,
    job_id VARCHAR(100) NOT NULL UNIQUE,
    status VARCHAR(20) NOT NULL,  -- processing, ready, downloaded, expired, deleted
    total_queries INTEGER NOT NULL,
    total_sparql INTEGER NOT NULL,
    total_cypher INTEGER NOT NULL,
    export_type VARCHAR(20) NOT NULL,  -- manual, deletion
    s3_key VARCHAR(500),
    s3_url VARCHAR(1000),
    expires_at TIMESTAMPTZ NOT NULL,
    created_at TIMESTAMPTZ DEFAULT now(),
    downloaded_at TIMESTAMPTZ,
    download_count INTEGER DEFAULT 0,
    trace_id VARCHAR(255)
);
CREATE INDEX idx_query_exports_user ON query_exports(user_id);
CREATE INDEX idx_query_exports_expires ON query_exports(expires_at);
```

**S3/MinIO:**
```
s3://vedo-replica-store/exports/queries/{user_id}/{job_id}.zip
```

**manifest.json (в ZIP):**

```json
{
  "version": "1.0",
  "exported_at": "2026-07-16T10:00:00Z",
  "user_id": "user-123",
  "username": "ivanov",
  "total_queries": 42,
  "queries": [
    {
      "id": "query-001", "name": "Мои машины", "type": "sparql",
      "created_at": "2026-07-15T10:00:00Z", "is_public": false,
      "tags": ["cars"], "file": "sparql/query-001.sparql",
      "hash": "sha256:abc123..."
    }
  ]
}
```

### 3. Поток экспорта

```
1. POST /api/v1/user/queries/export { export_type: "manual"|"deletion" }
2. Сбор запросов: получение всех сохранённых SPARQL/CYPHER из БД
3. Генерация: manifest.json + .sparql/.cypher файлы + ZIP
4. Загрузка в S3/MinIO (TTL: 7 дней manual, 30 дней deletion)
5. Запись в query_exports (PostgreSQL)
6. ≤ 50 запросов → синхронный ответ с download_url
   > 50 запросов → асинхронно, email через SMTP-шлюз
7. Скачивание: GET .../download → проверка auth + expiry → ZIP
8. Автоудаление: ежедневный cleanup просроченных экспортов
```

### 4. Безопасность

| Мера | Реализация |
|------|------------|
| **Аутентификация** | Bearer JWT, только свои запросы (`job.user_id == user.id`) |
| **Временные ссылки** | S3 presigned URL с TTL 1 час |
| **Шифрование** | S3 AES-256 at rest, TLS 1.3 in transit |
| **Аудит** | Все экспорты и скачивания логируются (365 дней) |

### 5. Конфигурация

```yaml
query_export:
  retention:
    manual_days: 7       # согласовано с decommission-export
    deletion_days: 30    # согласовано с account-closure-retention
    audit_days: 365
  performance:
    sync_threshold: 50   # ≤ 50 запросов → синхронно
  storage:
    type: s3
    bucket: vedo-replica-store   # существующее хранилище реплик
    prefix: exports/queries/
    lifecycle_days: 30
  notifications:
    reminder_days_before_expiry: 2
```

### 6. CLI-команды

```bash
vedo-cli export status --job-id job-123
vedo-cli export delete --job-id job-123 --force
vedo-cli export cleanup --dry-run
vedo-cli export cleanup --apply
vedo-cli export list --user-id user-123
```

## Рассмотренные альтернативы

| Альтернатива | Причина отклонения |
|--------------|--------------------|
| Только удаление без экспорта | Нарушение GDPR Article 20; Google Reader синдром |
| Экспорт в виде одного JSON | Неудобно для пользователя; сложно импортировать в другие системы |
| Без hash-сумм | Невозможность проверки целостности |
| Без уведомлений | Пользователь не знает о готовности асинхронного экспорта |
| Только при удалении | Ограничение функциональности; экспорт должен быть доступен всегда |
| Бессрочные ссылки | Риск утечки данных; временные ссылки безопаснее |

## Последствия

**Положительные последствия:**

1. **GDPR-совместимость:** Право на переносимость данных (Article 20) для сохранённых запросов.
2. **Минимальное воздействие:** Используется существующая инфраструктура — PostgreSQL, S3/MinIO, SMTP-шлюз. Новый сервис не создаётся.
3. **Удобный формат:** ZIP с .sparql/.cypher файлами и manifest.json с hash-суммами.
4. **Безопасность:** Временные ссылки, шифрование, аудит, авторизация по user_id.
5. **Согласованность:** Сроки хранения (7/30 дней) согласованы с существующими политиками `decommission-export` и `account-closure-retention`.

**Отрицательные последствия:**

1. **Нагрузка на S3:** Хранение ZIP-архивов (смягчается TTL 7/30 дней).
2. **Асинхронная задержка:** Для > 50 запросов требуется ожидание (смягчается email-уведомлением).

**Риски:**

1. **Утечка через ссылки:** Временные S3 presigned URL могут быть перехвачены.
   - **Смягчение:** HTTPS, TTL 1 час для presigned URL, аудит скачиваний.
2. **Потеря до скачивания:** Экспорт может быть удалён автоматически.
   - **Смягчение:** Напоминание за 2 дня до истечения.

## Связанные ADR

- `ADR-DES.DATA.decommission-export-format-strategy` — Общий формат экспорта (Turtle, JSON-LD, JSON Lines)
- `ADR-DES.DATA.account-closure-retention-strategy` — 30-дневный период охлаждения
- `ADR-DES.API.write-idempotency-strategy` — Идемпотентность (повторный экспорт не дублирует)
- `ADR-DES.INFRA.otel-observability-strategy` — OpenTelemetry (trace_id для мониторинга)

## Чек-лист реализации

- [ ] Таблица `query_exports` в PostgreSQL
- [ ] API endpoints: POST /export, GET /status, GET /download
- [ ] Логика сбора запросов и генерации ZIP
- [ ] Интеграция с S3/MinIO (существующее хранилище реплик)
- [ ] Email-уведомления через SMTP-шлюз
- [ ] UI: кнопка «Экспортировать запросы» в настройках
- [ ] UI: опция экспорта при удалении аккаунта
- [ ] CLI-команды (`vedo-cli export`)
- [ ] Ежедневный cleanup-скрипт
- [ ] Документация User Guide

---
