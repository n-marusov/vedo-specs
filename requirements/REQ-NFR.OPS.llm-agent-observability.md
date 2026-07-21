# Специализированная observability для LLM-driven pipelines

| Атрибут | Значение |
|---------|----------|
| **ID** | REQ-NFR.OPS.llm-agent-observability |
| **Уровень** | NFR |
| **Атрибут качества** | Supportability |
| **Приоритет** | P1 |
| **Статус** | ЧЕРНОВИК |
| **Источник** | Принцип специализированной наблюдаемости для AI-агентов (cost per task, tool call success rate, semantic cycles) поверх стандартного `REQ-NFR.OPS.metrics` и `REQ-FUN.INTEGRATION.cost-calculation` |

---

## Назначение

Стандартные технические метрики (HTTP 200 OK, latency P95, count) не показывают реальное состояние LLM-пайплайна и финансовых/качественных потерь. Введение reasoning-aware моделей усложняет учёт: метрика «токенов в ответе» остаётся зелёной, но реальный счёт вырастает кратно за счёт скрытых reasoning tokens. Это требование добавляет специализированные метрики сверх существующих `REQ-NFR.OPS.metrics`, `REQ-FUN.INTEGRATION.cost-calculation` (cost per request) и `REQ-NFR.OPS.suggestion-analytics` (acceptance rate для подсказок).

## Требование

### 1. Cost Per Task

- Метрика стоимости считается не по одному LLM-вызову (это `cost-calculation`), а по **задаче** — сумме всех LLM-вызовов, включаемых в решение одной user task (doc-extract на документ, генерация одной sequence, обработка одного NL-запроса с refinement-циклом).
- Обязательное включение reasoning tokens (если провайдер их экспонирует) — иначе фактический счёт окажется скрытым.
- Хранится гранулярно по `(task_type, user_id, ontology_id, provider, model)`.

### 2. Tool Call Success Rate (per-tool)

- Успешность каждого инструмента, который LLM-компонент вызывает (парсеры, ApplySequence, validators, NL-translators), считается отдельно.
- Один «дырявый» tool, возвращающий пустые/невалидные результаты, не должен маскироваться общей «зелёной» метрикой пайплайна — он идентифицируется отдельно, чтобы агент не сжигал токены впустую в retry-loops.
- Метрики: `vedo_llm_tool_calls_total{tool, status}`, `vedo_llm_tool_success_ratio{tool}`.

### 3. Плохие циклы (semantic repetition detection)

- Детектор «плохих циклов»: агент многократно делает запросы семантически близко к одному и тому же (меняя формулировки), но не получает успеха. Технически — не ошибка, но деньги и контекст сгорают.
- Сигнал в метрику `vedo_llm_repetition_cycles_total{task_type, ontology_id}` + алерт P1 при срабатывании (см. `REQ-NFR.SECURITY.llm-excessive-agency-control`).
- Алгоритм: embedding-similarity в скользящем окне N последних вызовов в одном task.

### 4. Reasoning-token-aware cost

- Если провайдер (OpenAI, Anthropic с «extended thinking», Google Gemini 2.5) экспонирует reasoning_tokens отдельно от completion_tokens — это отдельная секция в cost-grain.
- Метрики: `vedo_llm_reasoning_tokens_total{provider, model, task_type}`; дрейф reasoning ratio выше baseline × 2 → алерт.

### 5. Chain-level (не request-level) наблюдаемость

- Trace_id сквозит по всей task-цепочке (LLM → tool → LLM → tool), не обрывается на каждом компонентном вызове (OpenTelemetry propagation, см. `REQ-NFR.OPS.observability-stack`).
- Span per LLM-вызов + span per tool-call, объединённые в один parent span `task`.
- Дашборд «task view»: один task → его цепочка вызовов, токенов, ошибок, итоговая стоимость.

### 6. Что НЕ считается выполнением

- Только per-request `cost-calculation` без агрегации по задаче (task).
- HTTP-метрики без reasoning-token детекции и без semantic-repetition detection.
- Trace без сквозного trace_id между LLM-call и tool-call spans.

## Критерии приёмки

1. Метрика `vedo_llm_task_cost_usd_total{task_type, provider, model, ontology_id}` доступна в Prometheus и включает reasoning_tokens.
2. Метрики `vedo_llm_tool_calls_total` / `vedo_llm_tool_success_ratio` разделены по каждому tool; на дашборде виден «дырявый» tool.
3. При ≥ K семантически-повторяющихся вызовах за M минут в одном task генерируется P1-алерт и счётчик `vedo_llm_repetition_cycles_total` инкрементируется.
4. OpenTelemetry trace_id сквозит по всей task-цепочке; span `parent=task` объединяет все LLM/tooL-вызовы; Grafana-view показывает task → chain.
5. Дашборд отличается от `REQ-NFR.OPS.suggestion-analytics` (качество конкретных подсказок) — он не дублирует acceptance rate, а показывает производительность/стоимость решения задачи.
6. CI-тест `REQ-NFR.PROCESS.preprod-release-gates` включает сценарий: simulator reasoning-token blow-up → алерт в pre-prod, no green metric по чистой per-request cost.