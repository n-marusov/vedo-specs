# ADR-IMPL.STACK.public-browse-api-rust-strategy

**Дата:** 2026-07-14
**Статус:** PROPOSED

## Контекст

Public Browse API — read-only публичный API для просмотра опубликованных онтологий. По своей сути это тот же Ontology Service, но с урезанным интерфейсом: без операций редактирования, только выборка и просмотр графовых данных.

Для Ontology Service уже принято решение `ADR-IMPL.STACK.ontology-rust-strategy` в пользу Rust, так как graph CRUD и OWL/RDF-операции являются CPU-bound и чувствительны к задержкам GC.

## Требование-источник

- [backend-service-stack.md](requirements/REQ-CON.STACK.backend-service-stack.md)
- `ADR-IMPL.STACK.ontology-rust-strategy`
- `ADR-IMPL.STACK.microservice-language-stack-strategy`

## Решение

Реализовать Public Browse API на Rust, в одном cargo workspace с Ontology Service и Versioning Service.

Обоснование:

1. **Public Browse API выполняет те же CPU-bound графовые операции, что и Ontology Service** — чтение узлов, связей, иерархий, поиск по графу. Это не лёгкий I/O-bound CRUD, а полноценные графовые traversal с потенциально большими наборами данных. Rust с zero-cost abstractions и отсутствием GC-пауз даёт предсказуемое время ответа для read-query, что критично для публичного API под высокой нагрузкой.

2. **Максимальное переиспользование кода** — Public Browse API должен разделять с Ontology Service доменные модели, типы графовых структур, протоколы сериализации (RDF/OWL/JSON-LD) и слой доступа к Neo4j (Bolt-драйвер). Go-реализация потребовала бы дублирования логики графовых преобразований на другом языке или введения межсервисного gRPC-прокси, что добавляет latency и точки отказа.

3. **Единый cargo workspace** — Ontology Service, Versioning Service и Public Browse API уже находятся в общем Rust workspace. Выделение Public Browse API на Go означает:
   - Потерю возможности `cargo test` для сквозной проверки целостности данных между сервисами
   - Две разные codebase, синхронизирующие одни и те же типы графов
   - Дополнительный toolchain и Docker-образ в CI/CD

4. **Rust для read-only публичного API — не overkill** — хотя read-only API может казаться простым I/O-bound сценарием, в контексте онтологий каждый запрос может включать сложные рекурсивные обходы графа (subclass hierarchy, property chains, SPARQL-подобные traversal). Rust даёт однозначные преимущества именно для таких payload.

## Рассмотренные альтернативы

| Альтернатива | Причина отклонения |
|--------------|--------------------|
| **Go** (текущее решение в ADR-IMPL.STACK.microservice-language-stack-strategy) | Go с GC может давать latency spikes на больших графовых запросах; дублирование кода с Ontology Service; необходимость поддержки двух языков для графовой логики |
| **GraphQL Federation через API Gateway** | Добавляет сетевой hop и latency; усложняет эксплуатацию; не решает проблему дублирования кода графовых запросов |
| **Node.js** | Недостаточная производительность для синхронных операций обхода графа |

## Последствия

**Положительные последствия:**
- Единый стек с Ontology Service и Versioning Service — максимальное переиспользование типов, моделей, драйверов и тестов
- Единый cargo workspace — общая сборка, линтинг, тестирование
- Предсказуемая latency без GC-пауз для публичных запросов
- Унификация CI/CD — один Rust-конвейер для трёх сервисов

**Отрицательные последствия:**
- Rust остаётся менее распространённым языком, чем Go — выше порог входа для новых участников команды
- Изменение решения для Public Browse API требует согласования и обновления уже принятого `ADR-IMPL.STACK.microservice-language-stack-strategy`

**Меры снижения рисков:**
- Public Browse API изначально проектируется как тонкая read-only обёртка над shared библиотеками Ontology Service — минимум собственного кода
- Переиспользовать уже существующие тесты ontology-service через cargo workspace
- Опубликовать shared-крейты (`ontology-core`, `neo4j-client`) внутри workspace для явного описания границ переиспользования

## Related ADRs

- `ADR-IMPL.STACK.ontology-rust-strategy`
- `ADR-IMPL.STACK.version-control-rust-strategy`
- `ADR-IMPL.STACK.microservice-language-stack-strategy`

---
