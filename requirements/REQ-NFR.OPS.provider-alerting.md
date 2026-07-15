# Алертинг при отказе LLM-провайдеров

| Атрибут | Значение |
|---------|----------|
| **ID** | REQ-NFR.OPS.provider-alerting |
| **Уровень** | NFR |
| **Атрибут качества** | Supportability |
| **Приоритет** | P0 |
| **Статус** | ЧЕРНОВИК |
| **Источник** | Уточнение стейкхолдера по вопросу 1.3 матрицы качества M2 |
| **Связанные ADR** | `ADR-DES.INFRA.critical-alerts-strategy` |
| **Связанные REQ** | `REQ-NFR.OPS.provider-health-checks` |

---

## Назначение

Определить правила алертинга при отказах и деградации LLM-провайдеров.

## Требование

Система должна создавать алерты при следующих событиях, связанных с LLM-провайдерами:

**Матрица алертов:**

| Событие | Severity | Каналы | Действие | Debounce |
|---------|----------|--------|----------|----------|
| Провайдер `unavailable` (health check = 0) | **P1** | PagerDuty, Slack #incidents | Уведомление L1/L2 поддержки | 5 мин |
| Провайдер `degraded` (latency > порог) | **P2** | Slack #alerts | Уведомление L1 поддержки | 10 мин |
| Произошёл failover (переключение) | **P2** | Slack #alerts | Информирование о факте переключения | Нет (каждое событие) |
| Восстановление провайдера (unavailable → healthy) | **P3** | Slack #alerts | Информационное уведомление | Нет (каждое событие) |
| Все провайдеры `unavailable` | **P0** | PagerDuty (эскалация), Slack #incidents, Email on-call | Немедленная эскалация на L2/L3 | 2 мин |

**Содержание алерта (шаблон):**

```
Title: [P1] LLM Provider Unavailable: {provider_name}
Severity: P1
Time: {timestamp}
Provider: {provider_name}
Status: unavailable
Reason: {http_error_timeout_rate_limit}
Duration: {duration_since_first_failure}
Affected: NL→OWL, NL Query, AI Suggestions
Action: Проверить status.{provider}.com, проверить API key, проверить сетевую доступность
Runbook: {link_to_runbook}
```

## Критерии приёмки

1. Алерты создаются при каждом событии из матрицы.
2. P0/P1 алерты отправляются в PagerDuty/Opsgenie с эскалацией согласно on-call rotation.
3. Алерты содержат: название провайдера, причину, время, продолжительность, ссылку на runbook.
4. Алерты не дублируются (debounce согласно матрице).
5. Информация о статусе провайдеров синхронизируется со статус-страницей (`status.vedo.example.com`).
6. При восстановлении провайдера алерт автоматически резолвится.
