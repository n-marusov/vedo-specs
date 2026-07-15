# Мониторинг состояния circuit breaker

| Атрибут | Значение |
|---------|----------|
| **ID** | REQ-NFR.OPS.circuit-breaker-monitoring |
| **Уровень** | NFR |
| **Атрибут качества** | Supportability |
| **Приоритет** | P0 |
| **Статус** | ЧЕРНОВИК |
| **Источник** | Уточнение стейкхолдера по вопросу 2.3 матрицы качества M2 |

---

## Назначение

Обеспечить полный мониторинг состояния circuit breaker через Prometheus-метрики, алерты и Grafana-дашборд.

## Требование

**Метрики Prometheus:**

| Метрика | Тип |
|---------|------|
| `sparql_circuit_breaker_state` (0=CLOSED, 1=OPEN, 2=HALF-OPEN) | Gauge |
| `sparql_circuit_breaker_transitions_total` | Counter |
| `sparql_request_latency_seconds` | Histogram |
| `sparql_requests_total{status}` | Counter |
| `sparql_rejected_requests_total` | Counter |

**Алерты:**

| Событие | Severity | Условие |
|---------|----------|---------|
| Переход в OPEN | P1 | Автоматический переход |
| Ручной переход в OPEN | P2 | Ручное открытие |
| Длительное OPEN (> 5 мин) | P1 | OPEN > 5 минут |
| Частые переходы (> 3 за 10 мин) | P2 | Флаппинг |

## Критерии приёмки

1. Все метрики доступны в Prometheus.
2. Алерты отправляются в PagerDuty/Slack.
3. Дашборд доступен в Grafana с цветовым индикатором состояния.
4. История переходов доступна для post-mortem анализа.
5. Метрики хранятся минимум 30 дней.
