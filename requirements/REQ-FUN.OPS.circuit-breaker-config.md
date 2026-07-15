# Конфигурация параметров circuit breaker

| Атрибут | Значение |
|---------|----------|
| **ID** | REQ-FUN.OPS.circuit-breaker-config |
| **Уровень** | FUN |
| **Атрибут качества** | Supportability |
| **Приоритет** | P1 |
| **Статус** | ЧЕРНОВИК |
| **Источник** | Уточнение стейкхолдера по вопросу 2.3 матрицы качества M2 |

---

## Назначение

Предоставить администратору возможность конфигурировать параметры circuit breaker через YAML-конфигурацию с поддержкой hot reload.

## Требование

```yaml
sparql_circuit_breaker:
  enabled: true
  latency_threshold_seconds: 5.0
  observation_window_seconds: 60
  min_requests_for_observation: 10
  half_open_period_seconds: 15
  recovery_success_count: 3
  reopen_error_count: 1
  manual_override_enabled: true
  alerting_enabled: true
  audit_enabled: true
```

## Критерии приёмки

1. Все параметры конфигурируются через YAML.
2. Изменения применяются без перезапуска (hot reload).
3. При невалидной конфигурации используется предыдущая версия.
4. Изменения логируются в audit.
5. Документация содержит описание всех параметров и их допустимых диапазонов.
