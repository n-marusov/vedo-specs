# C4 Architecture — VEDO Core

## Диаграммы компонентов

### Ticket Management & Feedback System

_Референс: `ADR-IMPL.OPS.ticket-management-system-architecture`, `ADR-IMPL.STACK.ticketing-language-strategy`._

```mermaid
C4Component
    title Компоненты — Ticket Management & Feedback System

    Container_Boundary(ticketing, "Ticket Management") {
        Component(ticket_api, "TicketApi", "Go/REST", "Создание, обновление, просмотр тикетов")
        Component(telemetry_listener, "TelemetryListener", "Go", "Автотикеты по алертам и логам")
        Component(classifier, "Classifier", "Python", "Категория, приоритет, дедупликация")
        Component(metadata_enricher, "MetadataEnricher", "Go", "Версия, окружение, trace_id, URL, user-agent")
        Component(dedup_engine, "DedupEngine", "Python/Rules", "Сигнатура + окно 24 часа")
        Component(sync_worker, "SyncWorker", "Go", "Очередь синхронизации и обработка вебхуков")
        Component(notifier, "Notifier", "Go", "Email и in-app уведомления")
        Component(escalation_router, "EscalationRouter", "Go", "Маршрутизация P0/P1 в Slack/PagerDuty")
        Component(audit_log, "AuditLog", "Go", "История изменений статусов и комментариев")
    }

    Container(api_gw, "API Gateway", "Go/gin", "Проксирование /support/tickets")
    ContainerDb(ticket_db, "Ticket DB", "PostgreSQL", "Тикеты, комментарии, статус синхронизации")
    System_Ext(gitlab, "GitLab Issues", "Внешняя система тикетов")
    System_Ext(smtp, "SMTP-шлюз", "Почтовая доставка")
    System_Ext(monitoring, "Grafana Stack", "Prometheus + Loki + Tempo")

    Rel(api_gw, ticket_api, "REST /support/tickets")
    Rel(ticket_api, metadata_enricher, "Обогащает тикет")
    Rel(ticket_api, classifier, "Запрос классификации")
    Rel(classifier, dedup_engine, "Вычисляет сигнатуру")
    Rel(dedup_engine, ticket_db, "Проверяет дубликаты", "SQL")
    Rel(ticket_api, ticket_db, "Создаёт/обновляет тикет", "SQL")

    Rel(monitoring, telemetry_listener, "Передаёт события", "Webhook/API")
    Rel(telemetry_listener, ticket_api, "Создаёт автотикеты", "Internal REST")

    Rel(ticket_api, sync_worker, "Публикует задачу синхронизации")
    Rel(sync_worker, ticket_db, "Читает/обновляет состояние", "SQL")
    Rel(sync_worker, gitlab, "Двусторонняя синхронизация", "HTTPS/Webhook")

    Rel(ticket_api, notifier, "Событие изменения статуса")
    Rel(notifier, smtp, "Отправляет email", "SMTP/TLS")
    Rel(ticket_api, escalation_router, "P0/P1 тикеты")
    Rel(ticket_api, audit_log, "Пишет историю изменений")
    Rel(audit_log, ticket_db, "Сохраняет аудит", "SQL")

    UpdateLayoutConfig($c4ShapeInRow="3", $c4BoundaryInRow="1")
```
