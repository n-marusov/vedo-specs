# ADR-DES.INFRA.otel-observability-strategy

**Дата:** 2026-05-09  
**Статус:** Принято

## Требование-источник
- [observability-stack.md](requirements/REQ-NFR.OPS.observability-stack.md)

## Решение

Принять OpenTelemetry (сбор трейсов, метрик, логов) с Grafana Stack (Prometheus для метрик, Loki для логов, Tempo для трейсов, Grafana для дашбордов) как обязательный observability-стек базовой поставки VEDO Core.

OpenTelemetry обеспечивает единый стандарт инструментирования для всех сервисов (Rust, Go, Python, TypeScript), нейтральный к поставщику экспорт в любую серверную часть и production-ready корреляцию метрик, логов и трейсов — все требования production-готовности закрываются одним кроссплатформенным фреймворком.

Альтернативные серверные части (Victoria Metrics, Jaeger, Datadog, New Relic, ELK) допустимы через OpenTelemetry exporters или Prometheus remote write как опция заказчика, но не заменяют базовую поставку; поддержка альтернатив — зона ответственности заказчика или отдельная услуга.

## Последствия

- Единый стандарт для всех сервисов
- Единые дашборды в Grafana
- Дополнительная инфраструктура (Collector, Prometheus, Tempo, Loki)
- Специфичные для заказчика серверные части observability интегрируются через OpenTelemetry exporters или Prometheus-compatible remote write

---
