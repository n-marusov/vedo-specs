# Emergency disable для security policy

| Атрибут | Значение |
|---------|----------|
| **ID** | REQ-NFR.SECURITY.emergency-policy-disable |
| **Уровень** | NFR |
| **Атрибут качества** | Reliability |
| **Приоритет** | P0 |
| **Статус** | УТВЕРЖДЕНО |
| **Источник** | — |
| **Критерии приёмки** | — |

---

## Что описывает

Определяет детерминированную процедуру аварийного отключения проблемной security policy с ограничением blast radius и контролем времени восстановления.

## Область действия

- WAF/policy engine rules, authorization policy bundles, API protection rules, request filters.
- Только production-инциденты P0/P1, где policy вызывает массовые отказы или блокирует легитимный трафик.

## SLA аварийного отключения

| Параметр | Требование |
|---|---|
| Время активации global disable | <= 120 секунд |
| Время подтверждения восстановления ключевых метрик | <= 5 минут |
| Время публикации инцидент-апдейта после disable | <= 10 минут |
| Доля успешных drill по emergency disable | 100% ежеквартально |

## Условия запуска

Emergency disable допускается, если:

- `5xx rate` > 5% в течение 2 минут после изменения policy;
- либо `auth failure rate` > baseline x3 в течение 5 минут;
- либо подтвержден массовый false block легитимных запросов.

## Управление доступом

- Исполнители: Incident Commander + Security Lead (двойное подтверждение).
- Техническая команда: `vedo-cli security policy disable --scope global --ticket <id>`.
- TTL emergency disable: <= 60 минут с обязательным review/rollback policy.

## Пост-условия

- Обязательный post-disable smoke suite (`auth`, `read`, `write`, `policy_audit`).
- Обязательный RCA с корректирующими действиями <= 5 рабочих дней.
- Повторное включение policy только после pre-prod валидации и canary-проверки.

## Бизнес-правила

- Emergency disable без ticket ID и incident ID запрещен.
- Длительный global disable > 60 минут требует эскалации до L5.
- Локальное (tenant-scoped) отключение предпочтительно, если оно устраняет инцидент без global blast radius.
