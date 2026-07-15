# Аудит failover событий

| Атрибут | Значение |
|---------|----------|
| **ID** | REQ-NFR.DATA.failover-audit |
| **Уровень** | NFR |
| **Атрибут качества** | Supportability |
| **Приоритет** | P0 |
| **Статус** | ЧЕРНОВИК |
| **Источник** | Уточнение стейкхолдера по вопросу 1.3 матрицы качества M2 |
| **Связанные REQ** | `REQ-NFR.INTEGRATION.automatic-failover`, `REQ-NFR.DATA.llm-audit` |

---

## Назначение

Обеспечить полный аудит всех событий failover LLM-провайдеров для post-mortem анализа, compliance и расчёта SLO.

## Требование

Каждое событие, связанное с изменением статуса LLM-провайдера или переключением (failover), логируется в audit-системе.

**Логируемые поля:**

| Поле | Тип | Описание |
|------|-----|----------|
| `id` | UUID | Уникальный идентификатор записи |
| `timestamp` | TIMESTAMPTZ | Время события |
| `event_type` | ENUM | `provider_unavailable` / `provider_degraded` / `failover_started` / `failover_completed` / `provider_recovered` / `all_providers_unavailable` / `all_providers_recovered` |
| `provider_name` | TEXT | Провайдер, к которому относится событие |
| `previous_provider` | TEXT | Предыдущий активный провайдер (для failover) |
| `new_provider` | TEXT | Новый активный провайдер (для failover) |
| `reason` | TEXT | Причина: `timeout`, `http_5xx`, `rate_limit_429`, `connection_refused`, `health_check_failure` |
| `health_check_details` | JSON | Детали последних health check'ов |
| `duration_ms` | INTEGER | Время переключения (для failover_started → failover_completed) |
| `trace_id` | TEXT | OpenTelemetry trace_id для корреляции |

**Запрещено логировать:**

- API keys, токены доступа.
- Полные URL эндпоинтов с query-параметрами.
- Персональные данные пользователей.

**Срок хранения:**

- 90 дней для operational логов.
- 365 дней для compliance (Enterprise).

**Отчётность:**

Администратор имеет доступ к отчёту по failover событиям за период (месяц, квартал):
- Количество failover'ов по провайдерам.
- Суммарное время недоступности каждого провайдера.
- SLO compliance: (total_time - downtime) / total_time * 100%.

## Критерии приёмки

1. Каждое событие из enum `event_type` создаёт audit-запись.
2. Все обязательные поля заполнены.
3. Логи хранятся минимум 90 дней (операционные), до 365 дней (compliance, Enterprise).
4. Поиск по `provider_name`, `event_type`, `timestamp` работает.
5. Отчёт по failover доступен администратору через UI или API.
6. SLO дашборд обновляется автоматически на основе audit-логов.
