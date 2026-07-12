# Runbook: DDoS Attack Response

| Атрибут | Значение |
|---------|----------|
| **ID** | REQ-FUN.INFRA.runbook-procedures |
| **Уровень** | FUN |
| **Атрибут качества** | Supportability |
| **Приоритет** | P1 |
| **Статус** | УТВЕРЖДЕНО |
| **Источник** | — |
| **Критерии приёмки** | — |

---

**Runbook ID:** SEC-004
**Owner:** SRE Lead + Security Lead  
**Backup role:** Senior SRE  
**Review cadence:** 3 мес  
**Drill period:** 6 мес  
**Severity:** P1  
**Связанные алерты:** DDoS L7 detection, Rate limiting violations spike, WebSocket connection flood, SPARQL endpoint overload, Single IP traffic anomaly  

## Цель и scope

Обеспечение непрерывности сервиса при DDoS-атаке на VEDO Core. Runbook покрывает этапы подтверждения, mitigation, эскалации и post-mortem для атак на уровнях L3/L4 и L7.

**Scope:** API Gateway, Collaboration Service, SPARQL endpoint, WAF, cloud provider DDoS mitigation.

**Out of scope:** Атаки на инфраструктуру ниже API Gateway (cloud provider physical network), атаки на on-premise окружения заказчика.

## Диагностика

### Признаки атаки

| Сигнал | Источник | Порог |
|--------|----------|-------|
| Резкий рост 5xx ошибок | Gateway metrics | > 5% за 1 минуту |
| Алерт "Possible DDoS attack detected" | Prometheus Alertmanager | P1 |
| Rate limiting violations spike | Gateway metrics | > 1000 rate-limited за 1 минуту |
| SPARQL latency p99 > 30s | SPARQL metrics | При нагрузке > 60 req/min per IP |
| WebSocket connection flood | Collaboration Service metrics | > 1000 новых соединений/мин |
| Жалобы пользователей на недоступность | Support / Status page | — |

### Команды проверки

```bash
# Проверить общий RPS через API Gateway
vedo-cli diagnose metrics --query "sum(rate(api_gateway_requests_total[1m]))"

# Проверить 5xx rate
vedo-cli diagnose metrics --query "sum(rate(api_gateway_5xx_total[1m])) / sum(rate(api_gateway_requests_total[1m]))"

# Проверить rate limiting violations
vedo-cli diagnose metrics --query "sum(rate(api_gateway_rate_limited_total[1m]))"

# Проверить топ источников трафика (top 10 IP)
vedo-cli diagnose metrics --query "topk(10, sum by(client_ip) (rate(api_gateway_requests_total[5m])))"
```

## Восстановление

### Шаг 1: Подтвердить атаку

1. Проверить дашборд cloud provider (Yandex Cloud / AWS) на наличие DDoS-сигналов.
2. Проверить WAF dashboard на spike blocked requests.
3. Если cloud provider DDoS mitigation активирован — подтвердить, что L3/L4 защита работает.
4. Если cloud provider mitigation не сработал — открыть тикет поддержки провайдера.

### Шаг 2: Включить aggressive mode WAF

```bash
# Включить aggressive mode (блокировка по порогам 2x baseline)
vedo-cli emergency waf --mode aggressive
```

WAF aggressive mode:
- Блокировка по IP при > 200 req/min
- Блокировка по стране (если атака идёт из региона, где нет клиентов)
- Captcha challenge для подозрительных запросов

### Шаг 3: Усилить rate limiting (временная мера)

```yaml
# На время атаки (автоматически через API Gateway config update)
rate_limits:
  per_ip:
    unauthenticated: 20 req/min   # было 100
    authenticated: 200 req/min     # было 1000
  per_tenant: 1000 req/min         # было 5000
  sparql_per_ip: 10 req/min        # было 30
```

```bash
# Применить временные лимиты
vedo-cli emergency rate-limit --mode aggressive
```

### Шаг 4: При атаке на SPARQL endpoint

```bash
# Временно отключить SPARQL для неавторизованных
vedo-cli emergency kill --service sparql-public --duration 30m
```

### Шаг 5: При атаке на WebSocket (Commenting Service)

```bash
# Ограничить новые WebSocket-соединения
vedo-cli emergency rate-limit --websocket-max-connections 100
```

### Шаг 6: Уведомить клиентов

1. Обновить статус на status page: "Degraded performance — under DDoS attack."
2. Опубликовать сообщение в Slack/Teams канал клиентов.
3. Если атака длится > 15 минут — эскалировать.

## Верификация успеха

| Метрика | Целевое значение |
|---------|------------------|
| 5xx rate | < 1% за 2 минуты |
| P95 latency | < 5s (CRUD) / < 10s (SPARQL) |
| Rate limiting violations | < 100/min |
| WAF blocked requests | Стабильный или снижающийся |
| Доступность сервиса | 100% read, write по возможности |

```bash
# Проверить восстановление
vedo-cli diagnose metrics --query "sum(rate(api_gateway_5xx_total[2m])) / sum(rate(api_gateway_requests_total[2m]))"
```

## Эскалация

| Уровень | Кому | Тайминг | Канал |
|---------|------|---------|-------|
| L1 | Security Lead | В течение 15 минут после подтверждения | PagerDuty + Slack `#incident-p1` |
| L2 | SRE Lead + Platform Team | В течение 30 минут | Slack `#incident-p1` |
| L3 | Cloud provider support | При активации cloud mitigation или L3/L4 атаке | Тикет поддержки |
| L4 | Engineering Lead | При длительности > 1 час | Slack `#incident-p1` |

## RTO/RPO цели

- **RTO:** 30 минут (от подтверждения атаки до mitigation)
- **RPO:** N/A (DDoS не влияет на данные)

## Post-mortem

Обязательные пункты post-mortem после DDoS-инцидента:

1. **Анализ логов WAF и API Gateway** — определение вектора атаки (L3/L4/L7, целевые endpoint'ы, источники).
2. **Оценка эффективности mitigation** — время активации, достаточность порогов.
3. **Корректировка WAF-правил** — добавление сигнатур атаки в ruleset.
4. **Обновление runbook** — инцидент-специфичные шаги.
5. **Уведомление клиентов** — причинный анализ, corrective actions.

## Ownership и review

- Owner: SRE Lead + Security Lead
- Backup role: Senior SRE
- Review cadence: 3 месяца
- Drill period: 6 месяцев
- Last review: —
- Last drill: —

## Обязательные метрики evidence drill

| Метрика | Описание |
|---------|----------|
| `mitigation_activation_time` | Время от алерта до включения aggressive mode |
| `availability_during_attack` | Процент доступности сервиса во время атаки |
| `waf_blocked_total` | Количество заблокированных WAF запросов |
| `rate_limited_total` | Количество rate-limited запросов |
| `time_to_resolve` | Время от алерта до полного восстановления |
