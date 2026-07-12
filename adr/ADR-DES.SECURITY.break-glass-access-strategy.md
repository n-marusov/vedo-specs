# ADR-DES.SECURITY.break-glass-access-strategy

**Дата:** 2026-05-13  
**Статус:** Принято

## Контекст

Отказ внешнего Identity Provider или Keycloak может заблокировать администраторов именно тогда, когда требуется восстановление. Физический доступ не всегда доступен в SaaS или managed Kubernetes.

## Требование-источник
- [support-sla.md](requirements/REQ-NFR.SUP.support-sla.md)
- [deployment-model.md](requirements/REQ-CON.INFRA.deployment-model.md)
- [air-gapped-deployment.md](requirements/REQ-CON.INFRA.air-gapped-deployment.md)
- [runbook-ownership-drill-evidence.md](requirements/REQ-NFR.INFRA.runbook-ownership-drill-evidence.md)
- [security-requirements.md](requirements/REQ-NFR.SECURITY.security-requirements.md) — хранение Shamir-частей Emergency Admin пароля
- [UC-admin.backup.manage-backup-and-restore-via-cli](use-cases.md) — Emergency Admin L1: `vedo-cli restore`, `vedo-cli backup verify`
- [UC-diagnostics.trace.diagnose-incident-by-trace-id](use-cases.md) — Emergency Admin L1: `vedo-cli diagnose`

## Решение

Ввести три уровня break-glass доступа: L1 — Emergency Admin вне Keycloak с паролем, разделённым через Shamir Secret Sharing; L2 — offline recovery key для временного JWT; L3 — physical console или kubectl exec.

Трёхуровневая эскалация гарантирует, что отказ IdP или Keycloak не блокирует восстановление — Emergency Admin через независимую точку доступа /emergency/login сохраняет доступ к критическим операциям (emergency readonly, restore, diagnose) даже при полной недоступности основного провайдера аутентификации. Shamir M-of-N splitting, immutable audit log и немедленные уведомления security-команды предотвращают злоупотребление emergency-доступом.

Endpoint /emergency/login не зависит от Keycloak; возможности Emergency Admin ограничены emergency readonly, restore, diagnose и управлением admin-учётками; пароль ротируется каждые 90 дней и после каждого использования; L2/L3 — escalation при компрометации или недоступности L1.

## Рассмотренные альтернативы

| Альтернатива | Причина отклонения |
|--------------|--------------------|
| Только физический доступ | Недоступен или слишком медленный в SaaS/cloud сценариях |
| Emergency Admin в Keycloak | Не работает при отказе Keycloak/IdP |
| Неограниченный emergency admin | Слишком высокий риск злоупотребления |
| MFA через IdP для emergency path | Невозможно при отказе IdP; компенсируется Shamir splitting и audit |

## Последствия

**Положительные последствия:**
- Отказ IdP не блокирует восстановление.
- Emergency path ограничен критическими операциями и хорошо аудируется.
- Поддерживает SaaS, on-premise и air-gapped модели.

**Отрицательные последствия:**
- Нужно безопасно хранить части секрета и recovery key.
- Emergency path требует регулярной ротации и drill.

**Меры снижения рисков:**
 - Shamir M-of-N, immutable audit и немедленные уведомления.
 - Break-glass account test входит в operational drills.

---
