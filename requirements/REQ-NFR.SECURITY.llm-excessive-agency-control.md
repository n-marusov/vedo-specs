# Управление автономией LLM-компонентов и контроль Excessive Agency

| Атрибут | Значение |
|---------|----------|
| **ID** | REQ-NFR.SECURITY.llm-excessive-agency-control |
| **Уровень** | NFR |
| **Атрибут качества** | Security |
| **Приоритет** | P1 |
| **Статус** | ЧЕРНОВИК |
| **Источник** | Принцип Vulnerability = Autonomy × Authority (Don Song), контроль excessive agency |

---

## Назначение

Чем больше автономии и полномочий у LLM-компонента, тем строже должен быть мониторинг, аудит и возможность вмешательства. Автономия без контроля приводит к катастрофам (агент «почистить почту» с высокими полномочиями и высокой автономией, без контроля, удалил критичные данные).

В VEDO формула применяется так: если LLM-компонент получает право делать много шагов (multi-call chains, retry-loops, nested tool-calls), система мониторинга обязана охватывать целые цепочки задач, а не только технические метрики HTTP. См. `REQ-FUN.API.max-refinement-iterations`, `REQ-NFR.OPS.llm-agent-observability`.

## Требование

### 1. Bounded autonomy в коде

| Параметр | Требование |
|---|---|
| Maximun итераций LLM-вызова на одну задачу (task) | Жёсткий лимит в коде (≈ N итераций, конкретное значение согласно `max-refinement-iterations`); при превышении — task завершается ошибкой, без последующих попыток |
| Maximun число retry одной и той же операции LLM подряд (без изменения контекста) | Жёсткий лимит (по умолчанию 3); при превышении — alert + остановка chain |
| Maximun длительность одной LLM-задачи (wall-clock) | Жёсткий timeout, по умолчанию ≤ 120 сек на одну задачу; продление явным решением |
| Nested tool-call (LLM → tool → LLM → ...) | Запрещено по умолчанию; допускается только при явном allow-list в политике (см. `REQ-NFR.SECURITY.llm-tool-least-privilege`) |
| Автоматическое оздоровление (LLM сама ретраит проблему) | Запрещено для destructive-операций; разрешено только для read-only запросов с лимитом |

### 2. Контроль за runaway chains

- Мониторинг детектирует «плохие циклы» (semantic repetition: агент многократно ищет одно и то же, меняя формулировки — формально не ошибка, но деньги и контекст сгорают). См. `REQ-NFR.OPS.llm-agent-observability`.
- При детекции → автоматическая остановка chain + alert + audit-запись.

### 3. Авторитет относительно полномочий

- Политика `Vulnerability = Autonomy × Authority`: каждая LLM-функция оценивается по паре (autonomy score, authority score). При высокой authority (write/dataset-mod) допустима только низкая autonomy; высокая autonomy — только для низко-authority функций (read).
- Оценка зафиксирована в реестре LLM-функций (см. `REQ-NFR.SECURITY.llm-write-human-approval`, критерий 1).

### 4. Что НЕ считается выполнением

- Только лимит токенов, без лимита итераций / wall-clock / retry-chain.
- Pipeline метрик HTTP 200 OK без semantic / cost / chain analysis (см. `REQ-NFR.OPS.llm-agent-observability`).

## Критерии приёмки

1. В каждой LLM-функции начальные значения (max iter, max retry-same-context, wall-clock timeout, nested call allow) заданы в конфиге, отдельно от промпта; изменения логируются в audit.
2. Превышение лимитов завершает task явной ошибкой `E_LLM_EXCESSIVE_AGENCY`, а не продолжает chain.
3. Runaway-chain (>= N одинаковых или семантически-повторяющихся вызовов за M минут) детектируется и прерывается автоматически (см. `REQ-NFR.OPS.llm-agent-observability`).
4. Реестр LLM-функций содержит (autonomy × authority) оценку; для write/dataset-модификации артифактов установлен высокий authority → низкая autonomy enforced.
5. Тест `REQ-NFR.PROCESS.e2e-testing` содержит кейс «chain LLM-вызовов превышает лимит → graceful halt, без побочных эффектов».