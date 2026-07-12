# ADR-DES.INFRA.support-metadata-isolation-strategy

**Дата:** 2026-05-16  
**Статус:** Принято

## Контекст

Support metadata — данные, необходимые команде поддержки для идентификации tenant, связи с администратором, определения SLA, получения audit trail и выполнения break-glass процедуры — в текущей архитектуре VEDO Core не имеют единого изолированного хранилища. Tenant identity, контакты, SLA tier и deployment config overrides распределены по tenant DB (Neo4j, PostgreSQL Version Store), а audit trail критических операций, emergency keys и compliance evidence не имеют формальной политики хранения.

Это создаёт критическую уязвимость: при повреждении, удалении или недоступности tenant data команда поддержки теряет возможность идентифицировать tenant, найти контакты администраторов, определить SLA tier и escalation chain, получить audit trail и выполнить break-glass процедуру.

Индустриальные инциденты подтверждают реальность угрозы:
- **GitLab 2017:** При удалении production БД выяснилось, что метаданные клиентов (контакты админов, список tenant, backup credentials) хранились в той же БД. Восстановление заняло 18 часов, часть данных была потеряна.
- **Code Spaces 2014:** Злоумышленник получил доступ к AWS консоли и удалил все данные, включая metadata клиентов. Компания прекратила существование — не было изолированного хранилища для восстановления.

Для VEDO Core эта проблема усугубляется требованиями: SLA tiers с различными response time (Enterprise P0 — 15 мин), compliance (GDPR Art. 30, 152-ФЗ, SOC2), мультирегиональная архитектура и air-gapped развёртывания.

## Требование-источник
- Support Metadata Isolation Specification — `human/milestones/001-init/artifacts/support-metadata-isolation.md`
- [ADR-DES.INFRA.recovery-objectives-mandate](#adr-desinfrarecovery-objectives-mandate)
- [ADR-DES.INFRA.backup-policy-strategy](#adr-desinfrabackup-policy-strategy)
- [ADR-DES.SECURITY.break-glass-access-strategy](#adr-dessecuritybreak-glass-access-strategy)
- [ADR-DES.DATA.account-closure-retention-strategy](#adr-desdataaccount-closure-retention-strategy)
- [ADR-DES.OPS.support-sla-and-escalation-strategy](#adr-desopssupport-sla-and-escalation-strategy)

## Решение

Принять принцип изоляции support metadata от tenant storage — четыре типа хранилищ (Support DB, WORM S3, Vault/KMS, опционально ITSM) с раздельными retention, RTO и access control, не зависящих от работоспособности tenant DB.

**Категории данных и хранилища:**

| Категория | Хранилище | RTO | Retention | Шифрование |
|-----------|-----------|-----|-----------|------------|
| Tenant identity | Support DB (отдельный PostgreSQL) | 5 мин | Бессрочно + 1 год после удаления | AES-256 at rest |
| Contact information | Support DB | 5 мин | Бессрочно + 3 года | AES-256 at rest (PII-поля) |
| SLA tier | Support DB | 5 мин | Бессрочно + 3 года | AES-256 at rest |
| Audit trail (critical ops) | WORM S3 (Object Lock) | 15 мин | 7 лет | SSE-S3 / SSE-KMS |
| Billing metadata | Support DB | 15 мин | 5 лет | AES-256 at rest |
| Incident history | Support DB + ITSM (optional) | 15 мин | 3 года | AES-256 at rest |
| Escalation contacts | Support DB + PagerDuty | 5 мин | Пока активен контракт | AES-256 at rest |
| Emergency access keys | Vault / KMS | 10 мин | Пока tenant существует | Server-side encryption |
| Config overrides | Support DB + Git | 15 мин | Пока активен контракт | — |
| Compliance evidence | WORM S3 + Support DB (index) | 30 мин | 5 лет после закрытия tenant | SSE-S3 / SSE-KMS |

**Архитектурные требования:**

- **Support DB:** PostgreSQL 16, отдельный кластер от Version Store. Multi-AZ (2+ узла), синхронная репликация. Шифрование at rest (AES-256). IAM роли + mTLS для доступа. PITR backup (30 дней).
- **WORM S3:** S3 / MinIO с Object Lock в режиме COMPLIANCE. Retention 7 лет без возможности удаления или изменения. Cross-region репликация. Отдельные IAM credentials, запрет delete/overwrite.
- **Vault / KMS:** HashiCorp Vault (Enterprise) или AWS KMS / Yandex KMS. Active HA кластер. Auto-unseal через внешний KMS. Audit log всех операций. AppRole + OIDC federation.
- **ITSM (опционально):** Jira Service Management / Zendesk. Fallback на локальное хранение ticket summary в Support DB.

**Интерфейсы доступа:**

Support API (internal gRPC/REST, только из support VPN, не публичный). CLI через `vedo-cli support`:

| Команда | Описание | Роли |
|---------|----------|------|
| `vedo-cli support tenant-info <id>` | Полная информация о tenant | SRE, Support Engineer |
| `vedo-cli support audit-trail <id>` | Audit trail tenant | SRE, Support Engineer, Security Lead |
| `vedo-cli support list-backups <id>` | Список backup tenant | SRE, Support Engineer |
| `vedo-cli support emergency-access request <id>` | Запрос break-glass ключа | Security Lead (approval) |

Права доступа: SRE — полный доступ, кроме emergency access (только чтение audit). Support Engineer — чтение tenant identity, contacts, SLA, incidents; запись contacts и incidents. Security Lead — audit trail, emergency access, compliance evidence. Product Owner — billing, compliance.

**On-premise / air-gapped:** Заказчик предоставляет отдельную PostgreSQL. MinIO с Object Lock для WORM. Vault заменяется на encFS + key management. `vedo-cli support` работает полностью offline.

Детальная реализация описана в технической спецификации `human/milestones/001-init/artifacts/support-metadata-isolation.md`.

## Рассмотренные альтернативы

| Альтернатива | Причина отклонения |
|--------------|--------------------|
| **А: Всё в tenant DB** (текущее состояние) | При отказе tenant DB все support metadata недоступны. Инцидент GitLab 2017 показал, что восстановление без изолированной metadata занимает часы вместо минут. Невозможно выполнить break-glass без доступа к tenant DB. Противоречит compliance (audit trail должен быть доступен независимо) |
| **Б: Всё в отдельную support DB, без WORM и Vault** | Audit trail хранится в обычной БД и может быть случайно или злонамеренно изменён/удалён. Отсутствие WORM-защиты нарушает SOC2 CC6.1 (log integrity). Emergency keys в БД — риск компрометации. Vault предоставляет audit log доступа к ключам |
| **В: Внешняя CRM/ITSM как единственное хранилище** | CRM/ITSM не гарантируют RTO 5 мин при отказе tenant DB. Зависимость от внешнего сервиса (vendor lock-in). Нет WORM-хранилища для audit trail. Не поддерживает on-premise/air-gapped |
| **Г: Кэш support metadata в Redis с репликацией** | Redis — in-memory хранилище, данные теряются при перезапуске. Нет retention policy (данные должны храниться годами). Нет WORM-защиты. Не подходит для долгосрочного compliance-хранения |
| **Д: Cloud-native managed сервисы (AWS Aurora + S3 + Secrets Manager)** | Не подходят для on-premise и air-gapped сред (VEDO Core поддерживает Yandex Cloud, on-premise, air-gapped). AWS Secrets Manager не имеет on-premise аналога. Vendor lock-in |

## Последствия

**Положительные последствия:**
- Support metadata доступна при полной недоступности tenant DB — RTO 5 мин для контактов и SLA
- Audit trail защищён от случайного или злонамеренного удаления (WORM Object Lock, 7 лет)
- Emergency keys изолированы в Vault с полным audit log доступа и ротацией
- Compliance-соответствие: GDPR Art. 30, 152-ФЗ, SOC2 CC6.1
- Единый интерфейс доступа через `vedo-cli support`
- Поддержка on-premise и air-gapped сред

**Отрицательные последствия:**
- Дополнительная инфраструктура: отдельный PostgreSQL кластер, WORM bucket, Vault
- Увеличение стоимости: ≈ +$600–900/мес для SaaS
- Операционная сложность: обслуживание ещё одной БД
- Риск рассинхронизации между Support DB и tenant DB

**Меры снижения рисков:**

| Риск | Мера |
|------|------|
| Рассинхронизация Support DB и tenant DB | Автоматическая синхронизация при создании/удалении tenant (transactional outbox). Ежечасная проверка |
| Отказ Support DB | Multi-AZ (RTO 5 мин). PITR backup (30 дней). Restore drill ежемесячно |
| Компрометация WORM | Отдельные IAM роли. Запрет delete/overwrite. Аудит всех write-операций |
| On-premise — нет Vault | encFS + Shamir split (3 из 5). Физический доступ + 2 из 3 фрагментов |
| Устаревание contacts | Ежеквартальная верификация. Уведомление при смене администратора |

## Compliance Mapping

| Стандарт | Требование | Реализация |
|----------|------------|------------|
| **GDPR Art. 30** | Records of processing — хранение записей об обработке ПДн | Audit trail в WORM (7 лет). Tenant identity + contacts в Support DB (шифрование at rest) |
| **152-ФЗ ст. 21** | Журналы событий — фиксация действий с ПДн | Audit trail (WORM, 7 лет). Audit логов всех операций доступа к support metadata |
| **SOC2 CC6.1** | Log integrity — защита логов от модификации | WORM Object Lock (COMPLIANCE). Vault audit log в отдельном WORM bucket |
| **SOC2 A1.2** | Availability — доступность компонентов | Support DB Multi-AZ (RTO 5 мин). Мониторинг каждые 15 сек |

## Ссылки
- [Support Metadata Isolation Specification v1.0](human/milestones/001-init/artifacts/support-metadata-isolation.md) — техническая спецификация реализации
- [ADR-DES.INFRA.recovery-objectives-mandate](#adr-desinfrarecovery-objectives-mandate) — целевые RTO/RPO
- [ADR-DES.INFRA.backup-policy-strategy](#adr-desinfrabackup-policy-strategy) — политика резервного копирования
- [ADR-DES.SECURITY.break-glass-access-strategy](#adr-dessecuritybreak-glass-access-strategy) — процедура emergency доступа
- [ADR-DES.DATA.account-closure-retention-strategy](#adr-desdataaccount-closure-retention-strategy) — политика закрытия tenant
- [ADR-DES.OPS.support-sla-and-escalation-strategy](#adr-desopssupport-sla-and-escalation-strategy) — SLA и матрица эскалации
- `human/constraints/security.yaml` — глобальные ограничения безопасности

---
