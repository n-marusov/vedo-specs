# Мониторинг здоровья LLM-провайдеров

| Атрибут | Значение |
|---------|----------|
| **ID** | REQ-NFR.OPS.provider-health-checks |
| **Уровень** | NFR |
| **Атрибут качества** | Supportability |
| **Приоритет** | P0 |
| **Статус** | ЧЕРНОВИК |
| **Источник** | Уточнение стейкхолдера по вопросу 1.3 матрицы качества M2 |
| **Связанные ADR** | `ADR-DES.INFRA.otel-observability-strategy` |
| **Связанные REQ** | `REQ-NFR.INTEGRATION.automatic-failover`, `REQ-NFR.OPS.provider-alerting` |

---

## Назначение

Определить параметры мониторинга здоровья и доступности LLM-провайдеров через health checks.

## Требование

Система должна непрерывно мониторить доступность и производительность всех настроенных LLM-провайдеров через периодические health checks. Результаты используются для принятия решения о failover.

**Параметры health check:**

| Параметр | Значение |
|----------|----------|
| **Интервал проверки** | 30 секунд |
| **Метод проверки** | HTTP HEAD или GET к `/v1/models` (OpenAI/Anthropic compatible API) |
| **Тайм-аут проверки** | 5 секунд |
| **Порог детекции отказа** | 3 последовательных неудачных проверки |
| **Порог восстановления** | 2 последовательных успешных проверки |

**Статусы провайдера:**

| Статус | Код | Критерий |
|--------|-----|----------|
| `healthy` | 2 | Все health checks успешны |
| `degraded` | 1 | HTTP 429 (rate limit) или p95 latency > 10 сек |
| `unavailable` | 0 | HTTP ошибка (5xx, timeout) или 3 последовательных неудачи |

**Метрики Prometheus:**

```
llm_provider_status{provider="openai"}           # 0/1/2
llm_provider_latency_p95{provider="openai"}      # seconds
llm_provider_latency_p99{provider="openai"}      # seconds
llm_provider_errors_total{provider="openai"}      # counter
llm_provider_failover_count{provider="openai"}     # counter
llm_provider_health_check_success{provider}       # 0/1 per check
```

## Критерии приёмки

1. Health checks выполняются каждые 30 секунд для каждого сконфигурированного провайдера.
2. Метрики экспортируются в Prometheus и отображаются в Grafana.
3. Алерт создаётся при переходе провайдера в статус `unavailable` (см. `REQ-NFR.OPS.provider-alerting`).
4. Алерт создаётся при p95 latency > 10 секунд в течение 5-минутного окна.
5. Дашборд «M2 AI & Query Health» показывает статус всех провайдеров в реальном времени.
6. Для локальной LLM (on-premise) health check также выполняется с теми же параметрами.
