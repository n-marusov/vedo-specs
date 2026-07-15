# Конфигурация тарифов LLM-провайдеров

| Атрибут | Значение |
|---------|----------|
| **ID** | REQ-FUN.OPS.provider-rate-config |
| **Уровень** | FUN |
| **Атрибут качества** | Supportability |
| **Приоритет** | P0 |
| **Статус** | ЧЕРНОВИК |
| **Источник** | Мульти-провайдерная модель VEDO |

---

## Назначение

Предоставить администратору возможность конфигурировать тарифы LLM-провайдеров для расчёта стоимости AI-запросов.

## Требование

Администратор может настраивать тарифы LLM-провайдеров в системе. Тарифы используются для расчёта стоимости AI-запросов и списания AI Credits.

**Формат конфигурации:**

```yaml
llm_providers:
  openai:
    models:
      gpt-4o:
        input_price_per_1m: 5.00
        output_price_per_1m: 15.00
        cache_discount: 0.5
      gpt-4:
        input_price_per_1m: 30.00
        output_price_per_1m: 60.00
        cache_discount: 0.5
      gpt-4o-mini:
        input_price_per_1m: 0.15
        output_price_per_1m: 0.60
        cache_discount: 0.5
  anthropic:
    models:
      claude-3-opus:
        input_price_per_1m: 15.00
        output_price_per_1m: 75.00
        cache_discount: 0.5
      claude-3-sonnet:
        input_price_per_1m: 3.00
        output_price_per_1m: 15.00
        cache_discount: 0.5
      claude-3-haiku:
        input_price_per_1m: 0.25
        output_price_per_1m: 1.25
        cache_discount: 0.5
  google:
    models:
      gemini-1.5-pro:
        input_price_per_1m: 2.50
        output_price_per_1m: 10.00
        cache_discount: 0.5
      gemini-1.5-flash:
        input_price_per_1m: 0.075
        output_price_per_1m: 0.30
        cache_discount: 0.5
  local:
    models:
      llama3-70b:
        input_price_per_1m: 0
        output_price_per_1m: 0
        cache_discount: 0.5
      qwen-72b:
        input_price_per_1m: 0
        output_price_per_1m: 0
        cache_discount: 0.5
```

## Критерии приёмки

1. Администратор может обновлять тарифы провайдеров через конфигурацию.
2. Изменение тарифов применяется без перезапуска сервисов (hot reload).
3. История изменений тарифов логируется в audit.
4. При изменении тарифов все новые запросы используют обновлённые цены.
5. Синхронизация с официальными тарифами провайдеров — документация содержит инструкцию.
