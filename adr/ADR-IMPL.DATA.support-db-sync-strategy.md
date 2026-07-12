# ADR-IMPL.DATA.support-db-sync-strategy

**Дата:** 2026-05-23  
**Статус:** Предложено

## Контекст

`support-metadata-isolation.md` фиксирует изоляцию Support DB от tenant DB, но не определяет реализуемую стратегию синхронизации метаданных tenant. Без отдельного решения остаются открытыми вопросы о механизме доставки изменений, регулярной сверке, обработке расхождений и восстановлении после сбоев.

## Требование-источник
- `human/artifacts/requirements/REQ-NFR.INFRA.support-metadata-isolation.md` (раздел `Synchronization Requirements`, R-01..R-08)
- [ADR-DES.INFRA.support-metadata-isolation-strategy](#adr-desinfrasupport-metadata-isolation-strategy)

## Решение

### A-01: Transactional Outbox для event-driven синхронизации
- Изменения tenant-метаданных (create tenant, delete tenant, change SLA, change owner) пишутся в `tenant_outbox` в той же транзакции, что и изменение в tenant DB.
- События имеют уникальный `event_id`; потребитель обязан быть идемпотентным.
- Механизм обеспечивает атомарность записи бизнес-данных и события, минимизируя риск потери изменений при частичных сбоях.

### A-02: CDC Worker для доставки в Support DB
- Отдельный `outbox-worker` (Go) читает `tenant_outbox` с polling-интервалом 100 мс.
- Воркер применяет изменения в Support DB и помечает событие как обработанное (`processed=true`, `processed_at`).
- Метрики синхронизации: `support_sync_lag_seconds`, `support_sync_events_total`, `support_sync_failures_total`.

### A-03: Periodic Reconciliation Job (daily)
- Ежедневный reconciliation job запускается в 02:00 UTC (Kubernetes CronJob).
- Джоб сравнивает tenant DB и Support DB по ключевым полям: `tenant_id`, `sla_tier`, `region`, `owner_id`.
- При расхождениях: однозначные исправляются автоматически, неоднозначные создают тикет для поддержки.

### A-04: Data Model for Sync Observability
- Вводятся таблицы `sync_metadata` и `sync_discrepancies` для аудита состояния синхронизации и диагностики отклонений.
- `sync_discrepancies` хранит: `tenant_id`, `discrepancy_type`, `expected_value`, `actual_value`, `resolved`, `resolved_at`, `resolved_by`.

```sql
CREATE TABLE tenant_outbox (
    id UUID PRIMARY KEY,
    event_type VARCHAR(64) NOT NULL,
    tenant_id UUID NOT NULL,
    payload JSONB NOT NULL,
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),
    processed BOOLEAN NOT NULL DEFAULT FALSE,
    processed_at TIMESTAMP NULL
);

CREATE TABLE sync_metadata (
    id BIGSERIAL PRIMARY KEY,
    sync_type VARCHAR(32) NOT NULL,
    last_sync_timestamp TIMESTAMP NOT NULL,
    status VARCHAR(32) NOT NULL,
    records_processed INT NOT NULL
);

CREATE TABLE sync_discrepancies (
    id BIGSERIAL PRIMARY KEY,
    tenant_id UUID NOT NULL,
    discrepancy_type VARCHAR(64) NOT NULL,
    expected_value JSONB NOT NULL,
    actual_value JSONB NOT NULL,
    resolved BOOLEAN NOT NULL DEFAULT FALSE,
    resolved_at TIMESTAMP NULL,
    resolved_by VARCHAR(128) NULL,
    created_at TIMESTAMP NOT NULL DEFAULT NOW()
);
```

## Рассмотренные альтернативы

| Альтернатива | Причина отклонения |
|--------------|--------------------|
| CDC с Debezium + Kafka | Избыточно для текущего этапа, добавляет операционную сложность и новую инфраструктуру |
| Двойная запись в коде (tenant DB + Support DB) | Нет атомарности, высокий риск частичных обновлений при сбое между операциями |
| Только event-driven без периодической сверки | Рассинхронизация может оставаться незамеченной при сбоях доставки или silent corruption |
| Только periodic без event-driven | Не выполняется SLA по критическим событиям (задержка до 24 часов) |

## Последствия

**Положительные последствия:**
- Критические изменения попадают в Support DB с минимальной задержкой, соответствующей R-04.
- Сверка раз в 24 часа обнаруживает и устраняет дрейф данных.
- Появляется прозрачная история рассинхронизаций и их разрешения через SQL/API.

**Отрицательные последствия:**
- Добавляется новый операционный компонент (`outbox-worker`) и дополнительные таблицы.
- Требуется поддержка идемпотентности и мониторинга lag/ошибок синхронизации.
- Periodic reconciliation создает дополнительную нагрузку на БД в окне выполнения.

**Меры снижения рисков:**
- Алертинг на `support_sync_lag_seconds` и наличие неразрешенных `sync_discrepancies`.
- Ограничение reconciliation по батчам и backoff при росте нагрузки.
- Runbook восстановления: запуск ускоренной сверки после восстановления tenant DB.

## Ссылки
- [ADR-DES.INFRA.support-metadata-isolation-strategy](#adr-desinfrasupport-metadata-isolation-strategy)
- `human/artifacts/requirements/REQ-NFR.INFRA.support-metadata-isolation.md`

---
