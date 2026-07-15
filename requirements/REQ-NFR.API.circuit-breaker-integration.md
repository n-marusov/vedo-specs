# Интеграция circuit breaker с API Gateway

| Атрибут | Значение |
|---------|----------|
| **ID** | REQ-NFR.API.circuit-breaker-integration |
| **Уровень** | NFR |
| **Атрибут качества** | Reliability |
| **Приоритет** | P0 |
| **Статус** | ЧЕРНОВИК |
| **Источник** | Уточнение стейкхолдера по вопросу 2.3 матрицы качества M2 |

---

## Назначение

Интегрировать circuit breaker с API Gateway для защиты всех запросов к SPARQL endpoint.

## Требование

Circuit breaker встроен в API Gateway и применяется ко всем запросам `/api/v1/sparql`.

**Поток обработки:**
1. Request → Check Circuit Breaker state
2. CLOSED → Forward to SPARQL Service
3. OPEN → Return HTTP 503
4. HALF-OPEN → Forward with counter (limiting concurrent probe requests)

**Сбор метрик:** Latency, request count, error count — для каждого запроса.

## Критерии приёмки

1. Circuit breaker применяется ко всем запросам к SPARQL endpoint.
2. В OPEN возвращается HTTP 503.
3. В HALF-OPEN пропускается ограниченное число запросов.
4. Метрики собираются для всех запросов без исключения.
5. Интеграция добавляет задержку < 1 мс.
