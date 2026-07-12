# ADR-IMPL.STACK.microservice-language-stack-strategy

**Дата:** 2026-05-19
**Статус:** PROPOSED → ACCEPTED

## Контекст

В VEDO Core уже зафиксированы частные ADR по языкам для части сервисов:
- `ADR-IMPL.STACK.ontology-rust-strategy`
- `ADR-IMPL.STACK.version-control-rust-strategy`
- `ADR-IMPL.STACK.auth-service-go-strategy`
- `ADR-IMPL.STACK.metrics-python-strategy`
- `ADR-IMPL.STACK.frontend-vue-strategy`

Для API Gateway, Ticket Service, Publisher Service, Public Browse API и Commenting Service язык реализации явно не был закреплён отдельным ADR, что блокировало старт реализации stubs и унификацию CI/CD-конвейеров.

Требуется единый ADR верхнего уровня, который фиксирует целостную language/runtime стратегию для всех микросервисов VEDO Core и сохраняет согласованность с уже принятыми решениями.

## Решение

Принять единый принцип выбора стека:
- CPU-bound сервисы: Rust
- I/O-bound сервисы с большим количеством соединений: Go
- I/O-bound CRUD/integration сервисы: Go
- Аналитика/ETL: Python
- Frontend: Vue 3 + TypeScript

Назначить язык/рантайм для всех сервисов VEDO Core:

| Сервис | Язык / рантайм | Обоснование |
|--------|-----------------|-------------|
| Ontology Service | Rust (tokio) | CPU-bound операции над графом; уже закреплено в `ADR-IMPL.STACK.ontology-rust-strategy` |
| Versioning Service | Rust (tokio) | compute-intensive diff/merge; уже закреплено в `ADR-IMPL.STACK.version-control-rust-strategy` |
| Auth Service | Go | high-throughput stateless auth; уже закреплено в `ADR-IMPL.STACK.auth-service-go-strategy` |
| Metrics Service | Python (FastAPI) | analytics/aggregation; уже закреплено в `ADR-IMPL.STACK.metrics-python-strategy` |
| API Gateway | Go | I/O-bound proxy/middleware, высокая конкуррентность соединений, gRPC/REST/GraphQL маршрутизация |
| Ticket Service | Go | CRUD + внешние REST-интеграции (Jira/YouTrack), PostgreSQL, быстрый delivery |
| Publisher Service | Rust | критична CPU-bound часть экспорта графа и формирования больших snapshot-файлов |
| Public Browse API | Go | read-only публичный API с высоким трафиком, простое горизонтальное масштабирование |
| Commenting Service | Go | WebSocket + Redis Pub/Sub, много долгоживущих соединений |
| Frontend | TypeScript + Vue 3 | уже закреплено в `ADR-IMPL.STACK.frontend-vue-strategy` |

Выборы для новых сервисов выполнены по аналогии с уже принятыми ADR и соответствуют утверждённым принципам CPU-bound → Rust, I/O-bound → Go, analytics → Python, frontend → TS/Vue.

## Рассмотренные альтернативы

| Альтернатива | Причина отклонения |
|--------------|--------------------|
| Единый язык Go для всех сервисов | Упрощает унификацию, но теряется преимущество Rust в CPU-bound сервисах (Ontology, Versioning, Publisher) |
| Единый язык Rust для всех сервисов | Увеличивает сложность и стоимость разработки I/O-bound CRUD/integration сервисов; не покрывает аналитический контур так эффективно, как Python |
| Node.js для API Gateway и Commenting | Подходит для I/O, но для высоконагруженных WebSocket/proxy сценариев выбран Go (пропускная способность, эксплуатационная предсказуемость, модель goroutines) |

## Последствия

**Положительные последствия:**
- Оптимальный язык под профиль нагрузки каждого сервиса
- Единообразие и совместимость с существующими ADR
- Снижение неопределённости для реализации stubs и CI/CD bootstrap
- Использование сильных сторон Rust, Go, Python и TypeScript/Vue по назначению

**Отрицательные последствия:**
- Полиглотность усложняет CI/CD (разные toolchains, сборки, базовые образы)
- Повышенные требования к экспертизе команды в нескольких языках

**Меры снижения рисков:**
- Унифицированные Docker-образы по стеку (Rust/Go/Python/Node)
- Стандартизированные `make`-цели для каждого сервиса (`lint`, `test`, `build`, `docker-build`)
- GitHub Codespaces/devcontainer с предустановленными toolchains
- Шаблоны CI-пайплайнов для повторного использования между сервисами

## Related ADRs

- `ADR-IMPL.STACK.ontology-rust-strategy`
- `ADR-IMPL.STACK.version-control-rust-strategy`
- `ADR-IMPL.STACK.auth-service-go-strategy`
- `ADR-IMPL.STACK.metrics-python-strategy`
- `ADR-IMPL.STACK.frontend-vue-strategy`

---
