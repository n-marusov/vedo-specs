# ADR-DES.DATA.storage-stack-strategy

**Дата:** 2026-05-09  
**Статус:** Принято

## Требование-источник
- [data-storage-stack.md](requirements/REQ-FUN.DATA.storage-stack.md)

## Решение

Принять стек хранения: Neo4j для графового хранилища (Triple Store), PostgreSQL + JSONB для Version Store, Redis Cluster для кэша и блокировок, RabbitMQ для очередей сообщений.

Neo4j обеспечивает производительность графовых операций с поддержкой Cypher и корпоративной кластеризации. PostgreSQL + JSONB даёт надёжное реляционное хранение с версионированием через JSONB-колонки. Redis Cluster обеспечивает TTL-кэширование и распределённые блокировки с низкой задержкой. RabbitMQ предоставляет проверенный event-driven конвейер для уведомлений и фоновых задач.

Собственный код VEDO Core распространять под MIT. Production-профили, требующие Neo4j Enterprise Cluster, использовать с BYOL/лицензией заказчика. Базовый MIT-дистрибутив не должен требовать функций Neo4j Enterprise (кластеризация, HA/failover, enterprise backup).

Version Store на PostgreSQL хранит не полные слепки онтологии, а дельты изменений на уровне триплетов. Каждый коммит содержит: `parent_commit_id`, `branch_id`, `message`, `timestamp`, `author_id`, `added_triplets` (JSONB), `removed_triplets` (JSONB). Текущее состояние активной ветки материализуется в Neo4j для обеспечения производительности чтения (цель G1: < 1 сек). При переключении ветки или восстановлении из коммита состояние Neo4j обновляется путём применения дельт от ближайшего материализованного слепка.

---
