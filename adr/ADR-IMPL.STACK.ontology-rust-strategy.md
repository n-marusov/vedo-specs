# ADR-IMPL.STACK.ontology-rust-strategy

**Дата:** 2026-05-09
**Статус:** Принято

## Контекст

Ontology Service — CPU-intensive сервис для graph CRUD, OWL/RDF операций. Требуется максимальная производительность для работы с графами до 1M триплетов.

## Требование-источник
- [backend-service-stack.md](requirements/REQ-CON.STACK.backend-service-stack.md)

## Решение

Реализовать Ontology Service на Rust с tokio async runtime и async-graphql.

Rust обеспечивает zero-cost abstractions и предсказуемую latency без GC-пауз — критично для CPU-интенсивных операций CRUD над графами до 1M триплетов и синхронных OWL/RDF-операций, где Go с GC или Java с overhead памяти неприемлемы.

Использовать cargo workspace для shared tooling с Versioning Service; интегрироваться с Neo4j через встроенный Bolt-драйвер.

## Рассмотренные альтернативы

- **Go** — хорошая производительность, но GC может вызывать latency spikes при больших payload
- **Java/Spring Boot** — высокое потребление памяти, GC overhead
- **Node.js** — недостаточная производительность для синхронных graph операций

## Последствия

- Zero-cost abstractions, близость к metal
- Отсутствие GC — предсказуемая latency
- Сложнее введение в курс дела для команды
- Меньше библиотек для RDF/OWL (ограничены существующие Rust-экосистемы)

---
