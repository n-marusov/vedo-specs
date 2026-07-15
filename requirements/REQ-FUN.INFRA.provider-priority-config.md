# Конфигурация приоритетов LLM-провайдеров

| Атрибут | Значение |
|---------|----------|
| **ID** | REQ-FUN.INFRA.provider-priority-config |
| **Уровень** | FUN |
| **Атрибут качества** | Supportability |
| **Приоритет** | P0 |
| **Статус** | ЧЕРНОВИК |
| **Источник** | Уточнение стейкхолдера по вопросу 1.3 матрицы качества M2 |
| **Связанные REQ** | `REQ-NFR.INTEGRATION.automatic-failover` |

---

## Назначение

Предоставить администратору возможность конфигурировать приоритет LLM-провайдеров и порядок их использования при failover.

## Требование

Администратор может настраивать приоритет LLM-провайдеров через конфигурацию tenant. Приоритет определяет, какой провайдер используется как основной и в каком порядке происходит переключение при отказах.

**Конфигурация (YAML, пример для SaaS):**

```yaml
llm:
  providers:
    openai:
      enabled: true
      priority: 1           # наивысший приоритет — основной провайдер
      model: gpt-4
      endpoint: https://api.openai.com/v1
    anthropic:
      enabled: true
      priority: 2           # резервный провайдер
      model: claude-3-opus
      endpoint: https://api.anthropic.com/v1
    local:
      enabled: false        # не используется в SaaS
      priority: 3

  failover:
    enabled: true
    check_interval: 30s     # интервал health check
    failure_threshold: 3    # последовательных неудач для детекции отказа
    recovery_threshold: 2   # последовательных успехов для восстановления
    degraded_latency_threshold: 10s  # порог latency для статуса degraded

  policies:
    public:
      allowed_providers: [openai, anthropic]
      fallback_order: [openai, anthropic]
    internal:
      allowed_providers: [openai, anthropic]
      fallback_order: [openai, anthropic]
    private:
      allowed_providers: [openai, anthropic]
      fallback_order: [openai, anthropic]
```

**Конфигурация для On-premise:**

```yaml
llm:
  providers:
    local:
      enabled: true
      priority: 1
      endpoint: ${VEDO_LLM_LOCAL_ENDPOINT}
      model: ${VEDO_LLM_LOCAL_MODEL}
    openai:
      enabled: false
    anthropic:
      enabled: false

  failover:
    enabled: false          # не применимо — один провайдер
```

**Применение конфигурации:**

- Конфигурация загружается при старте сервиса.
- Изменения применяются без перезапуска (hot reload) с интервалом проверки файла конфигурации не более 60 секунд.
- Изменение конфигурации логируется в audit (`admin_id, timestamp, old_config_hash, new_config_hash`).

## Критерии приёмки

1. Администратор может задать приоритет провайдеров через конфигурационный файл.
2. При отказе основного провайдера (priority=1) система переключается на следующий по приоритету.
3. Конфигурация применяется без перезапуска сервисов (hot reload).
4. Изменение конфигурации логируется в audit с указанием администратора и хешей конфигурации.
5. Для on-premise конфигурация содержит только локальную LLM, failover отключён.
6. Некорректная конфигурация (например, два провайдера с одинаковым priority) вызывает ошибку валидации при загрузке.
