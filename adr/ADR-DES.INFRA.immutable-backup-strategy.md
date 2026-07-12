# ADR-DES.INFRA.immutable-backup-strategy

**Дата:** 2026-05-13  
**Статус:** Принято

## Контекст

Backup, доступный теми же credentials, что и production, может быть удалён злоумышленником или администратором вместе с рабочими данными. Code Spaces (2014) показал, что такая архитектура может уничтожить компанию за одну ночь.

## Требование-источник
- [backup-storage.md](requirements/REQ-NFR.DATA.backup-storage.md)
- [backup-retention.md](requirements/REQ-NFR.DATA.backup-retention.md)
- [secure-erase.md](requirements/REQ-NFR.DATA.secure-erase.md)

## Решение

Минимальное требование: хотя бы одна backup-копия хранится в отдельном S3/MinIO-compatible bucket в другом регионе с изолированными credentials и WORM/Object Lock защитой.

Изоляция backup от production credentials предотвращает повторение сценария Code Spaces — production-кластер имеет write-only доступ к backup bucket без delete permissions, управление bucket выполняется из отдельной administrative сети, Object Lock защищает от удаления версий. Backup в том же регионе без региональной копии не считается достаточным.

Production credentials: write-only к backup bucket, без delete/version-delete; управление из отдельной administrative сети; Object Lock/WORM обязателен; отдельный cloud account или cross-cloud copy — рекомендуемое усиление, не обязательное.

## Рассмотренные альтернативы

| Альтернатива | Причина отклонения |
|--------------|--------------------|
| Обязательный отдельный cloud account для всех | Слишком тяжело для малых SaaS/on-premise внедрений; отдельный bucket с WORM покрывает минимум риска |
| Только bucket versioning без Object Lock/защиты версий | Администратор с delete-version правами может уничтожить backup |
| Хранить backup в том же регионе | Не покрывает региональную аварию |
| Production credentials могут удалять backup | Повторяет Code Spaces failure mode |

## Последствия

**Положительные последствия:**
- Компрометация production не уничтожает backup.
- Минимальная модель применима для SaaS, on-premise, air-gapped через MinIO/WORM.
- Более строгие клиенты могут выбрать отдельный account/project.

**Отрицательные последствия:**
- Нужно управлять отдельными credentials и lifecycle policy.
- Restore может быть сложнее из-за изоляции storage.

**Меры снижения рисков:**
- Lifecycle deletion выполняется только из management plane с MFA и audit.
- Restore drills проверяют доступность immutable backup.

---
