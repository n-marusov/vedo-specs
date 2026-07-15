# Состав audit-логов для MCP-инструментов

| Атрибут | Значение |
|---------|----------|
| **ID** | REQ-FUN.INTEGRATION.audit-log-content |
| **Уровень** | FUN |
| **Атрибут качества** | Supportability |
| **Приоритет** | P0 |
| **Статус** | ЧЕРНОВИК |
| **Источник** | Уточнение стейкхолдера по вопросу 2.5 матрицы качества M2 |

---

## Назначение

Определить полный состав полей audit-записи для каждого вызова MCP-инструмента.

## Требование

Каждый вызов MCP-инструмента создаёт audit-запись со следующими полями:

| Поле | Тип | Описание |
|------|-----|----------|
| `timestamp` | datetime | Время вызова |
| `user_id` | UUID | Идентификатор пользователя |
| `tool_name` | string | `query-sparql` / `query-cypher` / `introspect-ontology` |
| `parameters` | JSON | Параметры вызова (без sensitive data) |
| `request_preview` | string | Полный NL-запрос |
| `response_truncated` | string | Результат, truncated до 500 символов |
| `response_full_url` | string | Ссылка на полный результат в отдельном хранилище |
| `status` | string | `success` / `error` |
| `duration_ms` | integer | Длительность выполнения |
| `trace_id` | string | OpenTelemetry trace ID |
| `error_message` | string | Сообщение об ошибке (если была) |
| `source` | string | `claude_desktop` / `cursor` / `direct_api` |
| `ip_address` | string | IP-адрес клиента (опционально) |

## Критерии приёмки

1. Каждый вызов MCP-инструмента создаёт запись со всеми обязательными полями.
2. `request_preview` содержит полный NL-запрос (не truncated).
3. `response_truncated` содержит результат до 500 символов.
4. `response_full_url` содержит ссылку на полный результат.
5. Параметры не содержат sensitive data (маскируются до записи).
