# ADR-DES.PROCESS.incident-response-slo-strategy

**Дата:** 2026-05-17  
**Статус:** Принято

## Контекст

Существующие артефакты описывали эскалацию и лимиты шума алертов, но не фиксировали единые измеримые SLO для MTTA/MTTR с gate-условиями на релизы. Это создавало риск неоднозначной оценки качества incident response.

## Требование-источник

- [incident-response-slo.md](requirements/REQ-FUN.PROCESS.incident-response-slo.md)
- [alert-fatigue.md](requirements/REQ-NFR.OPS.alert-fatigue.md)
- [escalation-matrix.md](requirements/REQ-FUN.PROCESS.escalation-matrix.md)

## Решение

Установить обязательные пороги MTTA/MTTR/false-positive для P0/P1, ввести rolling 30d контроль и release/operational gates при систематическом нарушении порогов.

## Рассмотренные альтернативы

- Фиксировать только эскалационные таймеры — отклонено (не отражает реальное время восстановления).
- Post-factum разбор без числовых порогов — отклонено (невозможен objective control).

## Последствия

- **Плюсы:** объективный контроль эффективности реагирования и восстановимости.
- **Минусы:** дополнительная дисциплина в сборе точных timestamps и отчетности.
- **Смягчение:** автоматический сбор событий инцидента и append-only формат отчетов.

---
