# ADR-DES.SECURITY.emergency-policy-disable-strategy

**Дата:** 2026-05-17  
**Статус:** Принято

## Контекст

Ошибочные security policy могут вызвать массовый false block легитимного трафика. Нужна детерминированная процедура аварийного отключения с минимальным временем реакции.

## Требование-источник

- [emergency-security-policy-disable.md](requirements/REQ-NFR.SECURITY.emergency-policy-disable.md)
- [critical-alerts.md](requirements/REQ-NFR.OPS.critical-alerts.md)

## Решение

Утвердить global emergency disable policy с SLA активации <= 120 секунд, двойным подтверждением (Incident Commander + Security Lead), TTL <= 60 минут и обязательной post-disable валидацией.

## Рассмотренные альтернативы

- Только сервисный kill switch без policy-disable — отклонено (не покрывает false block policy class).
- Emergency disable без двойного подтверждения — отклонено (риск злоупотребления).

## Последствия

- **Плюсы:** быстрое восстановление легитимного трафика при policy-инцидентах.
- **Минусы:** риск ослабления защиты при длительном disable.
- **Смягчение:** TTL, L5-эскалация при превышении и обязательный RCA.

---
