# Конфигурируемый чёрный список запрещённых фраз

| Атрибут | Значение |
|---------|----------|
| **ID** | REQ-FUN.OPS.prompt-blacklist-config |
| **Уровень** | FUN |
| **Атрибут качества** | Supportability |
| **Приоритет** | P1 |
| **Статус** | ЧЕРНОВИК |
| **Источник** | Уточнение стейкхолдера по вопросу 2.1 матрицы качества M2 |

---

## Назначение

Предоставить администратору возможность конфигурировать чёрный список запрещённых фраз и паттернов для защиты от промпт-инъекций.

## Требование

Администратор может настраивать чёрный список через конфигурацию. Изменения применяются без перезапуска сервисов (hot reload).

**Формат конфигурации:**

```yaml
prompt_injection_prevention:
  blacklist:
    phrases:
      - "ignore previous instructions"
      - "forget all constraints"
      - "you are now a"
      - "system prompt"
      - "output only json"
      - "disregard your system prompt"
      - "from now on, act as"
      - "ignore the above"
      - "reset your instructions"
      - "you are no longer"
    patterns:
      - "^ignore.*instructions$"
      - "^.*system prompt.*$"
      - "^.*act as.*$"
  max_json_nesting: 3
  max_xml_nesting: 2
  max_length: 2000
  block_on_any_match: true
  audit_enabled: true
```

## Критерии приёмки

1. Администратор может добавлять/удалять фразы в чёрном списке.
2. Администратор может добавлять regex-паттерны.
3. Изменения применяются без перезапуска (hot reload).
4. История изменений чёрного списка логируется в audit.
5. При невалидной конфигурации используется предыдущая версия.
