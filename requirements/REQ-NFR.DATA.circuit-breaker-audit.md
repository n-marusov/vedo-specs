# Аудит событий circuit breaker

| Атрибут | Значение |
|---------|----------|
| **ID** | REQ-NFR.DATA.circuit-breaker-audit |
| **Уровень** | NFR |
| **Атрибут качества** | Supportability |
| **Приоритет** | P0 |
| **Статус** | ЧЕРНОВИК |
| **Источник** | Уточнение стейкхолдера по вопросу 2.3 матрицы качества M2 |

---

## Назначение

Логировать все события circuit breaker в audit-системе для анализа инцидентов и post-mortem расследований.

## Требование

Каждое событие circuit breaker логируется с полями:

| Поле | Описание |
|------|----------|
| `timestamp` | Время события |
| `event_type` | transition / manual_override / alert |
| `from_state` | Предыдущее состояние (CLOSED/OPEN/HALF-OPEN) |
| `to_state` | Новое состояние |
| `trigger` | latency_threshold / manual / error_count / recovery |
| `latency_p95` | Текущая p95 latency (сек) |
| `request_count` | Запросов в окне наблюдения |
| `error_count` | Ошибок в окне наблюдения |
| `user_id` | При ручном переключении |
| `trace_id` | OpenTelemetry trace_id |

## Критерии приёмки

1. Каждый переход логируется со всеми полями.
2. Ручные переключения логируются с user_id.
3. Алерты логируются с причиной.
4. Логи хранятся минимум 90 дней.
5. Поиск по event_type, trigger, user_id работает.
