# ADR-IMPL.STACK.metrics-python-strategy

**Дата:** 2026-05-09
**Статус:** Принято

## Контекст

Metrics Service — analytics/aggregation сервис для сбора метрик. Не критичен latency, важна скорость разработки и богатая экосистема для data processing.

## Требование-источник
- [backend-service-stack.md](requirements/REQ-CON.STACK.backend-service-stack.md)

## Решение

Реализовать Metrics Service на Python с FastAPI, pandas и prometheus_client.

Python обеспечивает быструю разработку и богатую экосистему для data processing (pandas, numpy) — оптимально для сервиса сбора и агрегации метрик, где latency не критичен, а скорость появления новых аналитических отчётов важнее максимальной производительности.

Использовать FastAPI для HTTP-интерфейса, prometheus_client для совместимости с PromQL, pandas для агрегации данных; GIL не является проблемой для I/O-bound metrics collection.

## Рассмотренные альтернативы

- **Go** — хорош, но лишний шаблонный код для data processing
- **Rust** — overkill, медленная разработка для analytics
- **Java** — слишком heavyweight для metrics collection

## Последствия

- Быстрая разработка для non-critical сервиса
- Pandas для data aggregation
- PromQL совместимость через prometheus_client
- GIL не проблема для I/O-bound metrics collection

---
