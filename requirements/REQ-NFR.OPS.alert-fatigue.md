# Лимиты alert fatigue

| Атрибут | Значение |
|---------|----------|
| **ID** | REQ-NFR.OPS.alert-fatigue |
| **Уровень** | NFR |
| **Атрибут качества** | Supportability |
| **Приоритет** | P1 |
| **Статус** | УТВЕРЖДЕНО |
| **Источник** | — |
| **Критерии приёмки** | — |

---

## Допустимые лимиты

| Метрика | Лимит | Способ измерения |
|---|---|---|
| **P1+ alerts на on-call в день при normal operation** | ≤ 5 | Prometheus `alertmanager_alerts_total` за 24h |
| **P0 false-positive rate** | < 1% | `p0_false_positive_ratio` за rolling 30d |
| **P1 false-positive rate** | < 5% | `p1_false_positive_ratio` за rolling 30d |
| **P0 alerts на on-call в день при normal operation** | 0 | P0 — это critical outage, не норма |
| **Total alerts (P0-P3) на on-call в день** | ≤ 20 | Общее количество за 24h |
| **Non-actionable P1+ pages в месяц (цель)** | ≤ 10% | `alertmanager_alerts{severity="p0\|p1"}` + ручная аннотация `actionable=true/false` |
| **Non-actionable P1+ pages в месяц (предел)** | ≤ 20% | |
| **Actionable-only для on-call пробуждения** | Только P0/P1, требующие ручного вмешательства < 2 часов | Автоматически разрешаемые проблемы (CPU spike < 5 мин) — запрещены к пробуждению |

## Типы алертов, имеющие право будить on-call

Только P0 и P1, где требуется ручное вмешательство менее 2 часов. Запрещено будить для:
- Автоматически разрешаемых проблем (CPU spike < 5 минут)
- Плановых событий (backup start)
- Предупреждений о capacity, которые можно отложить до утра

## Обязанность SRE

Команда SRE еженедельно пересматривает алерты, деактивирует шумные, переводит неоперативные P1 в P2.

Целевые пороги MTTA/MTTR и release-gates по скорости реакции определены в `incident-response-slo.md`.

## Механизмы контроля

1. **Traffic threshold для latency alerts** — P0 latency alert активируется только при объёме ≥ 100 запросов/минуту (зафиксировано в `ADR-DES.INFRA.critical-alerts-strategy`).
2. **False-positive regression gate** — если `p0_false_positive_ratio` > 1%, следующий deploy должен включать фикс или явное risk acceptance.
3. **Еженедельный alert review** — команда просматривает топ алертов за неделю, деактивирует/переводит в P2 бесполезные сигналы.
4. **Post-incident alert tuning** — каждый P0/P1 incident завершается проверкой: сработал ли alert вовремя, был ли он полезен singleton сигналом, без дубликатов и ложных срабатываний.

## Определение normal operation

- Отсутствие плановых maintenance window
- Отсутствие идущего в данный момент развёртывания major version
- Стандартная рабочая нагрузка (не стресс-тест, не DDoS)
- Статус всех сервисов — healthy по readiness probes

## Неприемлемые состояния (требуют tuning)

- > 1 P0 alert ложно в неделю
- Повторяющиеся P1 alerts, которые не требуют немедленного реагирования (следует перевести в P2)
- Множественные дублирующие алерты на одно корневое событие (требуется alert deduplication)
- Любая on-call неделя с > 50 total alerts — приводит к десенситизации и риску пропуска критического инцидента
