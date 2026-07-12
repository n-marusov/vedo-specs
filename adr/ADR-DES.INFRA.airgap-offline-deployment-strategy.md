# ADR-DES.INFRA.airgap-offline-deployment-strategy

**Дата:** 2026-05-10  
**Статус:** Принято

## Контекст

Часть on-premise заказчиков VEDO Core может работать в полностью изолированной среде без доступа в интернет: государственные информационные системы, оборонная промышленность, финансовые системы с высоким уровнем защиты и закрытые ЦОД. VEDO Core распространяется под лицензией MIT, поэтому online license activation и license server не требуются. Основной вопрос air-gap — техническая установка и эксплуатация без исходящего доступа в интернет.

## Требование-источник
- [air-gapped-deployment.md](requirements/REQ-CON.INFRA.air-gapped-deployment.md)
- [deployment-tiers-reference-architecture.md](requirements/REQ-CON.INFRA.deployment-tiers.md)

## Решение

Принять поддержку air-gapped развёртывания для on-premise установок — runtime services VEDO Core не должны требовать internet access.

Air-gapped режим делает VEDO Core пригодным для государственных информационных систем, оборонной промышленности и закрытых ЦОД. MIT-лицензирование не требует online license activation и не блокирует offline operation. Runtime остаётся полностью работоспособным внутри security perimeter: backup, restore, мониторинг и аутентификация работают без внешних вызовов.

Air-gap installation выполнять через заранее подготовленные container images, local registry (Harbor, Nexus) и local Helm charts. Установить флаги `VEDO_OFFLINE_MODE=true`, `VEDO_DISABLE_TELEMETRY=true`, `VEDO_DISABLE_VERSION_CHECK=true`. Keycloak должен работать с локальным realm или LDAP, Prometheus и Grafana — внутри контура, внешние PagerDuty и Slack заменить на SMTP.

**Уточнение по deployment tiers (v1.0):**
- Air-gapped профиль поддерживается как минимум для `Standard`; `High` допустим при наличии multi-AZ/мультикластерного эквивалента внутри изолированного контура.
- `Premium` в air-gapped среде требует двух географически/инфраструктурно независимых площадок клиента и отдельного канала репликации внутри закрытого периметра.
- Если у клиента отсутствует второй регион/площадка, заявлять tier `Premium` недопустимо.

## Рассмотренные альтернативы

| Альтернатива | Причина отклонения |
|--------------|--------------------|
| Требовать online license server | Противоречит MIT distribution и делает air-gap невозможным |
| Поддерживать только connected on-premise | Не покрывает госсектор, оборонку и closed datacenters |
| Оставить Docker Hub / Helm repo как обязательные dependencies | Air-gap installation не сможет выполняться без интернета |
| Оставить telemetry/version checks включёнными | Создаёт outbound network attempts и нарушает security perimeter |
| Использовать только external alerting PagerDuty/Slack | Эти каналы недоступны без интернета |

## Последствия

**Положительные последствия:**
- VEDO Core пригоден для закрытых контуров и регулируемых on-premise заказчиков.
- MIT licensing не блокирует offline operation.
- Runtime не зависит от external APIs.
- Backup, restore, monitoring и auth работают внутри perimeter.

**Отрицательные последствия:**
- Installation и updates становятся ручными: images/charts нужно переносить физически.
- Нужно поддерживать air-gap documentation и scripts, например `vedo-cli airgap prepare`.
- External integrations, PagerDuty, Slack и online documentation недоступны.
- Заказчик должен поддерживать local registry и local infrastructure mirrors.

**Меры снижения рисков:**
- Поставлять `values-airgap.yaml` и documented installation flow.
- Поставлять scripts для подготовки image bundle: `vedo-cli airgap prepare`.
- Поддерживать offline restore flag `--no-internet-check`.
- Документировать SMTP alerting внутри perimeter.
- Предоставлять локальный пакет документации для air-gapped заказчиков.

---
