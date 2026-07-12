# SLO инцидент-реакции: MTTA, MTTR и качество алертов

| Атрибут | Значение |
|---------|----------|
| **ID** | REQ-FUN.PROCESS.incident-response-slo |
| **Уровень** | FUN |
| **Атрибут качества** | Supportability |
| **Приоритет** | P0 |
| **Статус** | УТВЕРЖДЕНО |
| **Источник** | — |
| **Критерии приёмки** | — |

---

## Что описывает

Фиксирует измеримые SLO/SLA для скорости реакции и восстановления при P0/P1 инцидентах, а также лимиты качества алертов.

## Термины и формулы

- **MTTA** = среднее время от `detected_at` до `acknowledged_at`.
- **MTTR** = среднее время от `detected_at` до `restored_at`.
- **False-positive rate** = ложные алерты / все алерты соответствующего приоритета за окно.

Все timestamps обязательны: `detected_at`, `acknowledged_at`, `escalated_at`, `diagnosed_at`, `workaround_at`, `restored_at`, `closed_at`.

## Целевые пороги

| Метрика | P0 | P1 | Окно |
|---|---:|---:|---|
| MTTA (цель) | <= 5 минут | <= 15 минут | rolling 30d |
| MTTA (жесткий предел) | <= 10 минут | <= 30 минут | per incident |
| MTTR (цель) | <= 60 минут | <= 240 минут | rolling 30d |
| MTTR (жесткий предел) | <= 120 минут | <= 480 минут | per incident |
| False-positive rate | < 1% | < 5% | rolling 30d |

## Release и operational gates

- Если MTTA P0 (rolling 30d) > 5 минут, требуется corrective action plan до следующего production major release.
- Если MTTR P0 (rolling 30d) > 60 минут 2 месяца подряд, release sign-off блокируется до закрытия remediation tasks.
- Если false-positive P0 > 1% или P1 > 5%, обязателен alert tuning в следующем sprint.

## Роль и владение

- Incident Commander владеет качеством MTTA/MTTR по инциденту.
- SRE Lead владеет false-positive ratio и качеством alert routing.
- Security Lead дополнительно подтверждает метрики для security incidents.

## Отчётность

- Еженедельный operational report обязателен (P0/P1 counts, MTTA, MTTR, false-positive).
- Ежемесячный management report обязателен с трендами и отклонениями от целей.
- Формат машинного отчета: `incident-slo-report.json` (append-only хранение >= 3 лет).

## Связанные артефакты

- `critical-alerts.md`
- `alert-fatigue.md`
- `escalation-matrix.md`
- `sla-recovery.md`

## Бизнес-правила

- Инцидент без полного набора timestamps считается дефектом процесса и не может быть закрыт.
- Любой P0/P1 инцидент без назначенного Incident Commander дольше 5 минут считается нарушением процесса.
- Любая ручная корректировка incident timestamps запрещена; исправления только append-only с причиной и автором.
