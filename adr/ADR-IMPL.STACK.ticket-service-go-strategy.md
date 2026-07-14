# ADR-IMPL.STACK.ticket-service-go-strategy

**Дата:** 2026-07-14
**Статус:** PROPOSED

## Контекст

Ticket Service (ticket-api) — центральный CRUD-сервис подсистемы тикетов VEDO Core. Отвечает за создание, чтение, обновление и закрытие тикетов, управление их жизненным циклом, интеграцию с внешними системами (Jira, YouTrack) через REST, а также хранение данных в PostgreSQL.

Требования:
- CRUD-операции с предсказуемой задержкой
- REST-интеграции с внешними трекерами
- Хранение в PostgreSQL с типовыми запросами
- Быстрый delivery и простота сопровождения

Подсистема тикетов включает также асинхронные и интеграционные сервисы (classifier, notifier, sync, telemetry-listener), что подробно описано в `ADR-IMPL.STACK.ticketing-language-strategy`.

## Требование-источник

- [backend-service-stack.md](requirements/REQ-CON.STACK.backend-service-stack.md)
- `ADR-IMPL.OPS.ticket-management-system-architecture`
- `ADR-IMPL.STACK.ticketing-language-strategy`
- `ADR-IMPL.STACK.microservice-language-stack-strategy`

## Решение

Реализовать Ticket Service на Go.

Go оптимален для I/O-bound CRUD-сервиса с типовыми HTTP + PostgreSQL операциями: быстрая компиляция, простой параллелизм через горутины, богатая экосистема для REST (gin, gorilla/mux, chi) и драйверов PostgreSQL (pgx, lib/pq). Низкий порог входа для команды и быстрый цикл разработки.

Выбор Go для Ticket Service также согласован с `ADR-IMPL.STACK.ticketing-language-strategy`, который закрепляет Go для прикладного и интеграционного контура подсистемы тикетов (API, синхронизация, уведомления, телеметрия), оставляя Python только для классификатора.

## Рассмотренные альтернативы

| Альтернатива | Причина отклонения |
|--------------|--------------------|
| **Rust** | Избыточная сложность для CRUD + REST-интеграций; более долгий цикл изменений; нет преимуществ CPU-bound производительности для данного профиля нагрузки |
| **Python** | Хуже предсказуемость задержек и потребления ресурсов для high-throughput CRUD API; выше операционные риски — подробнее в `ADR-IMPL.STACK.ticketing-language-strategy` |
| **Java/Spring Boot** | Избыточное потребление памяти и сложность конфигурации для сервиса данного масштаба |

## Последствия

**Положительные последствия:**
- Быстрая разработка и простое сопровождение CRUD-логики
- Легковесный runtime — эффективное использование ресурсов
- Единый стек с другими I/O-bound сервисами (Auth, Commenting, Notifier, Sync)
- Простая интеграция с PostgreSQL через зрелые Go-драйверы

**Отрицательные последствия:**
- Один язык на большинство сервисов подсистемы тикетов снижает диверсификацию, но повышает единообразие и упрощает ротацию команды

## Related ADRs

- `ADR-IMPL.STACK.ticketing-language-strategy`
- `ADR-IMPL.OPS.ticket-management-system-architecture`
- `ADR-IMPL.STACK.microservice-language-stack-strategy`

---
