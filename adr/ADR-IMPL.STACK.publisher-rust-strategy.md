# ADR-IMPL.STACK.publisher-rust-strategy

**Дата:** 2026-07-14
**Статус:** PROPOSED

## Контекст

Publisher Service — CPU-bound сервис для экспорта опубликованных онтологий в snapshot-файлы и подготовки данных для Public Browse API. Критичные операции:

- Формирование полных snapshot-файлов графа для публикации (RDF/OWL/JSON-LD)
- Сериализация больших графовых структур с миллионами триплетов
- Импорт снапшотов в изолированный read-only Neo4j для Public Browse API
- Работа с теми же графовыми моделями и структурами, что и Ontology Service

По `ADR-DES.INFRA.ontology-publishing`, Publisher Service создаёт снапшот и импортирует данные в изолированный read-only Neo4j; Browse UI обращается только к Public Browse API.

## Требование-источник

- [backend-service-stack.md](requirements/REQ-CON.STACK.backend-service-stack.md)
- `ADR-DES.INFRA.ontology-publishing`
- `ADR-IMPL.STACK.ontology-rust-strategy`
- `ADR-IMPL.STACK.microservice-language-stack-strategy`

## Решение

Реализовать Publisher Service на Rust, в едином cargo workspace с Ontology Service, Versioning Service и Public Browse API.

Обоснование:

1. **CPU-bound операции экспорта** — формирование snapshot-файлов требует интенсивной сериализации графов, перебора триплетов, построения RDF/OWL-представлений. Rust с zero-cost abstractions и отсутствием GC-пауз обеспечивает максимальную производительность для этих задач.

2. **Максимальное переиспользование кода** — Publisher Service работает с теми же графовыми типами, моделями и Neo4j-драйвером, что и Ontology Service. Rust-реализация в одном cargo workspace позволяет переиспользовать:
   - Доменные модели узлов, связей, иерархий
   - Слой доступа к Neo4j через Bolt-драйвер
   - Функции сериализации RDF/OWL/JSON-LD
   - Типы и протоколы обмена с Versioning Service

3. **Единый Rust workspace** — Publisher Service разделяет cargo workspace с Ontology Service, Versioning Service и Public Browse API, что даёт общую сборку, линтинг и тестирование, а также эффективное переиспользование shared-крейтов.

## Рассмотренные альтернативы

| Альтернатива | Причина отклонения |
|--------------|--------------------|
| **Go** | Go с GC может давать latency spikes при сериализации больших графов; потеря возможности переиспользовать типы и драйверы из Ontology Service; дублирование графовой логики |
| **Python** | Недостаточная производительность для CPU-bound сериализации миллионов триплетов |

## Последствия

**Положительные последствия:**
- Единый Rust-стек с Ontology Service — максимальное переиспользование типов, драйверов и тестов
- Высокая производительность экспорта и сериализации графов
- Предсказуемое потребление памяти без GC-пауз
- Общий cargo workspace — одна сборка, один toolchain

**Отрицательные последствия:**
- Rust требует более высокой квалификации команды по сравнению с Go

**Меры снижения рисков:**
- Publisher Service изначально проектируется как тонкая обёртка над shared библиотеками из Ontology Service
- Минимизировать собственный код — максимально переиспользовать крейты workspace
- Покрыть критичные пути экспорта нагрузочными тестами

## Related ADRs

- `ADR-IMPL.STACK.ontology-rust-strategy`
- `ADR-IMPL.STACK.public-browse-api-rust-strategy`
- `ADR-DES.INFRA.ontology-publishing`
- `ADR-IMPL.STACK.microservice-language-stack-strategy`

---
