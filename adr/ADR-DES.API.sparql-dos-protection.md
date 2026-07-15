# ADR-DES.API.sparql-dos-protection — Защита SPARQL endpoint от DoS-атак

**Статус:** ПРЕДЛОЖЕНО
**Дата:** 2026-07-15

## Контекст

VEDO Hub предоставляет SPARQL endpoint для запросов к графу знаний (F10). SPARQL-запросы могут быть дорогими по вычислительным ресурсам, особенно при использовании неограниченных property path (например, `rdfs:subClassOf*`).

**Проблема: DoS-атаки через дорогие запросы**

Злоумышленник может отправить синтаксически валидные, но вычислительно дорогие SPARQL-запросы:

```sparql
SELECT ?class WHERE { ?class rdfs:subClassOf* :Vehicle }
```

Такой запрос может обойти весь граф классов, вызвать деградацию Neo4j и привести к отказу в обслуживании для других пользователей.

**Известные инциденты:**
- Wikidata Query Service неоднократно страдала от DoS через дорогие SPARQL-запросы
- DBpedia public SPARQL endpoint подвергалась атакам через property path без ограничений

**Требуется** многоуровневая защита:
1. **Предварительная фильтрация:** Блокировка опасных паттернов до выполнения
2. **Circuit breaker:** Автоматическая защита при деградации
3. **Мониторинг и алертинг:** Обнаружение и реагирование на атаки

## Требование-источник

- `REQ-CON.API.property-path-limit`
- `REQ-CON.API.unbounded-path-block`
- `REQ-NFR.API.circuit-breaker`
- `REQ-FUN.OPS.manual-circuit-breaker`
- `REQ-USR.UI.circuit-breaker-feedback`
- `REQ-NFR.OPS.circuit-breaker-monitoring`
- `REQ-NFR.DATA.circuit-breaker-audit`
- `REQ-NFR.PROCESS.circuit-breaker-tests`
- `REQ-NFR.API.circuit-breaker-integration`
- `REQ-FUN.OPS.circuit-breaker-config`

## Решение

Внедрить **многоуровневую защиту SPARQL endpoint**:

### Уровень 1: Предварительная фильтрация запросов — блокировка опасных паттернов до выполнения.
### Уровень 2: Circuit breaker — автоматическая защита при деградации производительности.
### Уровень 3: Мониторинг и алертинг — обнаружение и реагирование на аномалии.

---

### 1. Архитектурная схема

```
┌─────────────────────────────────────────────────────────────────────────┐
│                          API Gateway                                   │
│  ┌───────────────────────────────────────────────────────────────────┐ │
│  │               Уровень 1: Предварительная фильтрация              │ │
│  │  1. Парсинг SPARQL → синтаксическая ошибка → блокировка         │ │
│  │  2. Проверка property path: длина > 10 hops → блокировка        │ │
│  │     `*` или `+` без LIMIT → блокировка                          │ │
│  │  3. Проверка query complexity: score > 1000 → блокировка        │ │
│  └───────────────────────────────────────────────────────────────────┘ │
│                                    │                                   │
│                                    ▼                                   │
│  ┌───────────────────────────────────────────────────────────────────┐ │
│  │               Уровень 2: Circuit Breaker                         │ │
│  │  CLOSED → выполняем запрос                                       │ │
│  │  OPEN → HTTP 503, сообщение пользователю                         │ │
│  │  HALF-OPEN → пропускаем ограниченное число запросов              │ │
│  │  Триггер: p95 latency > 5 сек в течение 60 сек → OPEN           │ │
│  │  Восстановление: 3 успешных запроса → CLOSED                    │ │
│  └───────────────────────────────────────────────────────────────────┘ │
│                                    │                                   │
│                                    ▼                                   │
│  ┌───────────────────────────────────────────────────────────────────┐ │
│  │               Уровень 3: Мониторинг & Алертинг                   │ │
│  │  Prometheus: circuit_breaker_state, latency histogram, counters  │ │
│  │  Алерты: OPEN → P1, OPEN > 5 мин → P1, частые переходы → P2    │ │
│  └───────────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                     SPARQL Service → Neo4j Cluster                     │
└─────────────────────────────────────────────────────────────────────────┘
```

---

### 2. Уровень 1: Предварительная фильтрация

**Алгоритм:**

```python
def filter_sparql_query(query: str) -> FilterResult:
    # Шаг 1: Парсинг SPARQL
    try:
        parsed = sparql_parser.parse(query)
    except ParseException as e:
        return FilterResult(decision="BLOCK", reason="syntax_error", details=str(e))

    # Шаг 2: Проверка property path
    for path in parsed.property_paths:
        if path.is_unbounded:  # * или + без ограничения
            return FilterResult(decision="BLOCK",
                reason="unbounded_property_path",
                details=f"Path '{path.text}' uses '*' or '+' without length limit")

        if path.length > config.max_property_path_length:
            return FilterResult(decision="BLOCK",
                reason="property_path_too_long",
                details=f"Path length {path.length} exceeds limit {config.max_property_path_length}")

        if path.uses_forbidden_pattern():
            return FilterResult(decision="BLOCK",
                reason="forbidden_property_path",
                details=f"Path uses forbidden pattern: {path.pattern}")

    # Шаг 3: Query complexity
    complexity = calculate_query_complexity(parsed)
    if complexity > config.max_query_complexity:
        return FilterResult(decision="BLOCK",
            reason="query_too_complex",
            details=f"Complexity {complexity} exceeds limit {config.max_query_complexity}")

    return FilterResult(decision="ACCEPT", reason="all_checks_passed")
```

**Конфигурация:**

```yaml
sparql_filter:
  max_property_path_length: 10
  forbidden_property_paths:
    - "rdfs:subClassOf*"
    - "owl:sameAs*"
    - "rdfs:subPropertyOf*"
    - ".*\\+.*"
  max_query_complexity: 1000
  complexity_weights:
    property_path: 100
    union: 50
    optional: 30
    filter: 20
    limit_missing: 200
```

---

### 3. Уровень 2: Circuit Breaker

**Диаграмма состояний:**

```
CLOSED ──(p95 > 5s for 60s)──▶ OPEN ──(15s elapsed)──▶ HALF-OPEN
   ▲                             │                          │
   │                             │                          │
   └──(3 successes)──────────────┘    ◀──(1 error)──────────┘
```

**Параметры:**

| Параметр | Значение |
|----------|----------|
| Порог открытия | p95 latency > 5 сек в течение 60 сек (≥ 10 запросов) |
| Half-open период | 15 секунд |
| Порог восстановления | 3 успешных запроса подряд |
| Порог повторного открытия | 1 ошибка в HALF-OPEN |

**Поведение по состояниям:**
- **CLOSED:** Все запросы выполняются, метрики собираются.
- **OPEN:** Все запросы возвращают HTTP 503 с сообщением «Сервис временно перегружен. Попробуйте позже.»
- **HALF-OPEN:** Пропускается до 3 пробных запросов для проверки восстановления.

### 4. Уровень 3: Мониторинг и алертинг

**Метрики Prometheus:**
- `sparql_circuit_breaker_state` (gauge: 0=CLOSED, 1=OPEN, 2=HALF-OPEN)
- `sparql_circuit_breaker_transitions_total{from_state,to_state,trigger}` (counter)
- `sparql_request_latency_seconds` (histogram, buckets: 0.1-60s)
- `sparql_requests_total{status}` (counter)
- `sparql_rejected_requests_total{reason}` (counter)

**Алерты:**

| Событие | Severity | Условие |
|---------|----------|---------|
| Переход в OPEN | P1 | `sparql_circuit_breaker_state == 1` |
| OPEN > 5 минут | P1 | Состояние OPEN дольше 5 минут |
| Частые переходы | P2 | > 3 переходов за 10 минут |
| Высокая latency | P2 | p95 > 3 сек (предупреждение до срабатывания) |

---

### 5. Ручное управление

**CLI:**
```bash
vedo-cli sparql circuit-breaker status
vedo-cli sparql circuit-breaker --state open
vedo-cli sparql circuit-breaker --state closed
```

**API:** `POST /api/v1/admin/sparql/circuit-breaker` с телом `{"state": "open"}`.

**UI:** Административная панель с индикатором состояния, метриками (p95 latency, request count) и кнопками управления.

---

### 6. Интеграция с API Gateway

Circuit breaker реализован как middleware в API Gateway:

```go
func CircuitBreakerMiddleware(cb *CircuitBreaker) gin.HandlerFunc {
    return func(c *gin.Context) {
        if cb.State() == State.OPEN {
            c.JSON(503, gin.H{
                "error": "Service temporarily overloaded. Please try again later.",
                "code": "SERVICE_UNAVAILABLE",
                "retry_after": "15",
            })
            c.Abort()
            return
        }
        start := time.Now()
        c.Next()
        cb.RecordRequest(time.Since(start), c.Writer.Status())
        cb.CheckAndUpdateState()
    }
}
```

### 7. Конфигурация

```yaml
sparql:
  filter:
    max_property_path_length: 10
    forbidden_property_paths: ["rdfs:subClassOf*", "owl:sameAs*", "rdfs:subPropertyOf*", ".*\\+.*"]
    max_query_complexity: 1000
  circuit_breaker:
    enabled: true
    latency_threshold_seconds: 5.0
    observation_window_seconds: 60
    min_requests_for_observation: 10
    half_open_period_seconds: 15
    recovery_success_count: 3
    reopen_error_count: 1
    manual_override_enabled: true
  monitoring:
    metrics_enabled: true
    alerting_enabled: true
    audit_enabled: true
    metrics_retention_days: 30
```

## Рассмотренные альтернативы

| Альтернатива | Причина отклонения |
|--------------|--------------------|
| Только предварительная фильтрация | Невозможно предсказать все дорогие запросы — нужна реактивная защита |
| Только circuit breaker | Срабатывает после проблемы; фильтрация предотвращает до возникновения |
| Rate limiting вместо circuit breaker | Один дорогой запрос может лечь всю систему независимо от лимитов |
| Только ручное управление | Требует 24/7 мониторинга; автоматика критична для ночных инцидентов |
| Отключить property path полностью | Ограничение функциональности; property path важен для легитимных запросов |
| Только логирование без блокировки | Отсутствие реальной защиты |

## Последствия

**Положительные последствия:**

1. **Многоуровневая защита:** Фильтрация предотвращает проблемы, circuit breaker реагирует на возникшие.
2. **Автоматическая защита:** Без ручного вмешательства — критично для ночных инцидентов.
3. **Прозрачность:** Пользователи видят понятные сообщения с причиной блокировки.
4. **Управляемость:** Администратор может переопределить автоматику через CLI/API/UI.
5. **Наблюдаемость:** Полный мониторинг (Prometheus + Grafana) и аудит всех событий.

**Отрицательные последствия:**

1. **Ложные срабатывания:** Легитимные запросы могут быть заблокированы фильтром.
2. **Сложность:** Дополнительный компонент circuit breaker увеличивает сложность системы.
3. **Задержка:** Фильтрация добавляет ≤ 10 мс к каждому запросу.
4. **Обход:** Злоумышленники могут найти способы DoS, не покрытые фильтрами.

**Меры снижения рисков:**

1. **Ложные срабатывания:** Чёткие сообщения о причине, возможность обращения в поддержку.
2. **Сложность:** Модульная архитектура, документация и тесты.
3. **Задержка:** Оптимизация — компилированные регулярные выражения, ранний выход.
4. **Обход:** Регулярное обновление фильтров на основе анализа инцидентов.

**Риски:**

1. **Недостаточное покрытие:** Новый тип атаки не попадёт в фильтр.
   - **Смягчение:** Circuit breaker как вторая линия защиты.
2. **Зацикливание (flapping):** Частые переходы OPEN ↔ HALF-OPEN.
   - **Смягчение:** Дебаунс через half-open с проверкой 3 успешных запросов подряд.
3. **Производительность:** Сбор метрик и проверка состояния добавляют задержку.
   - **Смягчение:** Асинхронный сбор метрик, lock-free структуры данных.

## Связанные ADR

- `ADR-DES.API.sparql-query-language-strategy` — SPARQL как язык запросов
- `ADR-DES.INFRA.fault-tolerance-strategy` — Отказоустойчивость
- `ADR-IMPL.SECURITY.parser-query-fuzz-gates-mandate` — Фаззинг-гейты парсеров
- `ADR-DES.INFRA.otel-observability-strategy` — OpenTelemetry стек
- `ADR-DES.INFRA.critical-alerts-strategy` — Критичные алерты

## Чек-лист реализации

- [ ] Реализация фильтрации SPARQL-запросов (парсинг, property path, complexity)
- [ ] Реализация circuit breaker (состояния, переходы, метрики)
- [ ] Интеграция с API Gateway (middleware)
- [ ] Настройка Prometheus-метрик и Grafana-дашборда
- [ ] Настройка алертов (P1/P2)
- [ ] UI для ручного управления circuit breaker
- [ ] CLI-команды (`vedo-cli sparql circuit-breaker`)
- [ ] CI/CD-тесты (6 сценариев)
- [ ] Документация Admin Guide: защита SPARQL endpoint
- [ ] Документация User Guide: ограничения property path

---
