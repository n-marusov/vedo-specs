# ADR-DES.DATA.config-overrides-retention-strategy

**Дата:** 2026-05-23  
**Статус:** Предложено

## Контекст

`support-metadata-isolation.md` фиксирует хранение deployment config overrides (Helm values, custom domain, SSL metadata, custom themes, feature flags), но не задает формальный retention после закрытия tenant, юридические исключения и процедуру автоматической очистки.

GDPR и 152-ФЗ не устанавливают отдельный прямой retention именно для config overrides как технического класса данных. Для этой категории политика выбирается на основе принципов минимизации данных, privacy-by-design, правовых исключений (legal hold, right to erasure) и операционных ожиданий Enterprise-клиентов по восстановлению.

## Требование-источник
- `human/artifacts/requirements/REQ-NFR.INFRA.support-metadata-isolation.md` (раздел `Deployment Config Overrides Retention`, R-01..R-08)
- [ADR-DES.DATA.account-closure-retention-strategy](#adr-desdataaccount-closure-retention-strategy)
- [ADR-DES.INFRA.support-metadata-isolation-strategy](#adr-desinfrasupport-metadata-isolation-strategy)

## Решение

### A-01: Retention period - 90 дней
- Базовый срок хранения overrides: 90 дней от `hard_deleted_at`.
- Цель: дать окно восстановления для Enterprise (1-3 месяца) без избыточного долгосрочного хранения.

### A-02: Хранилище - WORM S3 (Object Lock)
- Конфигурации и связанные артефакты хранятся в S3-совместимом WORM (Object Lock, режим GOVERNANCE).
- Срок блокировки: 90 дней или продление по legal hold.

### A-03: Механизм очистки - ежедневный CronJob + `vedo-cli purge-configs`
- Kubernetes CronJob запускается ежедневно в 02:00 UTC.
- Базовая команда: `vedo-cli purge-configs --older-than 90d`.
- Очистка удаляет метаданные в Support DB и данные в объектном хранилище для записей с истекшим retention.

### A-04: GDPR erasure - флаг `purge_immediately`
- В Support DB вводится флаг `purge_immediately BOOLEAN DEFAULT FALSE`.
- Для подтвержденного запроса по GDPR Art. 17 флаг устанавливается в `TRUE`; запись удаляется при ближайшем запуске purge job вне общей очереди 90 дней.

### A-05: Legal hold - таблица `legal_holds`
- Добавляется таблица удержаний и проверка при очистке:

```sql
CREATE TABLE legal_holds (
    id UUID PRIMARY KEY,
    tenant_id UUID NOT NULL,
    hold_until_date DATE NOT NULL,
    authorized_by VARCHAR(255) NOT NULL,
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),
    notes TEXT
);
```

- Эффективный retention рассчитывается как `MAX(90_days, hold_until_date + 90_days)`.

### A-06: 152-ФЗ обезличивание - удаление `custom_domain`
- Для tenant региона РФ при закрытии `custom_domain` очищается, остается только хэш для forensic:

```sql
UPDATE config_overrides
SET custom_domain = NULL,
    custom_domain_hash = encode(digest(custom_domain, 'sha256'), 'hex')
WHERE tenant_id = $1
  AND custom_domain IS NOT NULL;
```

- Это снижает риск прямой идентификации tenant через доменное имя при сохранении технической трассируемости.

## Рассмотренные альтернативы

| Альтернатива | Причина отклонения |
|--------------|--------------------|
| Retention 30 дней | Недостаточное окно восстановления для Enterprise-кейсов |
| Retention 365 дней | Избыточный срок хранения, повышенный риск утечки и конфликт с privacy-by-design |
| Обычный S3 без WORM | Нет гарантии неизменяемости до истечения retention |
| Очистка по event trigger | Сложнее операционно, нет требования real-time purge |
| Заморозка всего tenant при legal hold | Юридически обычно удерживаются конкретные категории данных, не весь tenant-контур |
| Шифрование `custom_domain` вместо удаления | Добавляет lifecycle ключей без выигрыша по рискам по сравнению с обезличиванием |

## Последствия

**Положительные последствия:**
- Политика retention становится юридически и операционно определенной.
- Есть управляемое окно восстановления конфигураций (90 дней).
- Снижается риск бесконтрольного накопления конфигураций удаленных tenant.
- WORM и audit trail дают проверяемость для compliance и расследований.

**Отрицательные последствия:**
- Увеличение объема хранения в S3 в пределах retention-окна.
- Дополнительная логика в purge job (GDPR override, legal hold, региональные правила).
- Требуется процесс подтверждения erasure-запросов и входящих legal hold.

**Меры снижения рисков:**
- Мониторинг количества/объема overrides и алерты на рост.
- Алерт при failed CronJob очистки.
- Юридический runbook для регистрации и снятия legal hold.

## Ссылки
- [ADR-DES.DATA.account-closure-retention-strategy](#adr-desdataaccount-closure-retention-strategy)
- [ADR-DES.INFRA.backup-policy-strategy](#adr-desinfrabackup-policy-strategy)
- [ADR-IMPL.STACK.vedo-cli-framework-strategy](#adr-implstackvedo-cli-framework-strategy)
- `human/artifacts/requirements/REQ-NFR.INFRA.support-metadata-isolation.md`

---
