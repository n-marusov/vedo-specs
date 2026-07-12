# ADR-DES.INFRA.critical-alerts-strategy

**Дата:** 2026-05-10  
**Статус:** Принято

## Контекст

VEDO Core должен быть production-ready для SaaS и on-premise развёртывания. Одного observability stack недостаточно: нужны заранее определённые критичные алерты, severity model, пороги реакции и граница ответственности для управляемой заказчиком серверной части observability.

## Требование-источник
- [critical-alerts.md](requirements/REQ-NFR.OPS.critical-alerts.md)

## Решение

Принять четырёхуровневую модель алертов (P0 Critical, P1 High, P2 Warning, P3 Info) с заранее определёнными порогами, каналами оповещения и лимитами усталости.

Заранее определённые алерты делают VEDO Core production-ready для SaaS и on-premise без необходимости настройки с нуля. Модель severity гарантирует, что P0 (потеря connectivity, OOM, риск потери данных, критическая задержка) получает немедленную реакцию, а P3 не создаёт шума. Лимиты усталости (P1+ ≤ 5 в день, доля ложных P0 < 1%) предотвращают alert fatigue. On-premise заказчик может менять пороги и каналы через конфигурацию Prometheus без изменения кода.

Настроить отправку P0/P1 в PagerDuty или Opsgenie, P0/P1/P2 в Slack или Teams, P2+ на email. OOM-события должны автоматически собирать heap dump. Еженедельный разбор алертов обязателен для деактивации шумных сигналов, перевода неоперативных P1 в P2 и дедупликации.

## DDoS Detection Alerts (P1)

Добавлены алерты для обнаружения DDoS-атак на уровнях L7:

```yaml
- name: DDoS L7 detection
  expr: |
    (
      sum(rate(api_gateway_requests_total[1m])) > 10000
      and
      sum(rate(api_gateway_5xx_total[1m])) > 0.2
    )
  labels:
    severity: p1
  annotations:
    summary: "Possible DDoS attack detected (L7)"
    runbook: "runbooks/ddos-attack-response.md"

- name: Rate limiting violations spike
  expr: |
    sum(rate(api_gateway_rate_limited_total[1m])) > 1000
  labels:
    severity: p1
  annotations:
    summary: "Rate limiting violations spike"
    runbook: "runbooks/ddos-attack-response.md"

- name: WebSocket connection flood
  expr: |
    sum(rate(websocket_connections_total[1m])) > 1000
  labels:
    severity: p1
  annotations:
    summary: "WebSocket connection flood detected"
    runbook: "runbooks/ddos-attack-response.md"
```

## Рассмотренные альтернативы

| Альтернатива | Причина отклонения |
|--------------|--------------------|
| Только базовые health checks | Не покрывает деградацию задержки, OOM, отставание репликации и риск потери данных |
| Только инфраструктурные алерты CPU/RAM/disk | Не отражает пользовательские SLO, долю ошибок и бизнес-критичные операции |
| Все алерты как P0 | Создаёт усталость от алертов и снижает качество реакции на настоящие инциденты |
| Настраивать алерты только на стороне заказчика | Неприемлемо для готовности базовой поставки VEDO Core к production |
| Алертить задержку без минимального порога нагрузки | Единичные выбросы создают ложные критические инциденты |

## Последствия

**Положительные последствия:**
- Production-развертывание получает минимальный набор заранее определённых критичных алертов.
- Реагирование на инциденты привязано к критичности и времени реакции: P0 15 минут, P1 1 час, P2 24 часа.
- Потеря связности, OOM, потеря данных и критическая задержка становятся явными требованиями готовности релиза.
- On-premise заказчик может менять пороги и каналы через конфигурацию Prometheus без изменения кода.

**Отрицательные последствия:**
- Нужно поддерживать правила Prometheus, дашборды Grafana и runbooks для разных моделей развертывания.
- Пороговые значения могут требовать настройки под реальные нагрузки корпоративных инсталляций.
- Интеграция PagerDuty, Opsgenie, Slack, Teams, email и status page добавляет эксплуатационные настройки.

**Меры снижения рисков:**
- Поставлять дефолтные Prometheus rules вместе с observability stack.
- Разрешить переопределения заказчика для on-premise через конфигурацию Prometheus.
- Для Datadog, New Relic и других альтернативных серверных частей требовать эквивалентные алерты как ответственность заказчика или отдельную платную интеграцию.
- Использовать порог нагрузки для алертов задержки, чтобы снизить число ложных срабатываний.

---
