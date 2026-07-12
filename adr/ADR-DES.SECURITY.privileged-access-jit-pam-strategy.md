# ADR-DES.SECURITY.privileged-access-jit-pam-strategy

**Дата:** 2026-05-17  
**Статус:** Принято

## Контекст

Компрометация привилегированных учетных записей остается основным вектором критических инцидентов. Standing access и слабые MFA-практики неприемлемы.

## Требование-источник

- [privileged-access-control.md](requirements/REQ-NFR.SECURITY.privileged-access-control.md)

## Решение

Запретить standing access в production, внедрить JIT/PAM с TTL <= 60 минут, 100% session recording и phishing-resistant MFA (FIDO2/WebAuthn) для всех привилегированных ролей.

## Рассмотренные альтернативы

- VPN + static admin accounts — отклонено (высокий риск lateral movement).
- TOTP-only MFA — отклонено (уязвимость к phishing/relay).

## Последствия

- **Плюсы:** резкое снижение риска компрометации privileged access.
- **Минусы:** рост операционной нагрузки на выдачу временных доступов.
- **Смягчение:** автоматизированный JIT workflow и шаблоны emergency approvals.

---
