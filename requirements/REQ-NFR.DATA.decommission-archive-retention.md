# Архивирование артефактов и retention при выводе из эксплуатации

| Атрибут | Значение |
|---------|----------|
| **ID** | REQ-NFR.DATA.decommission-archive-retention |
| **Уровень** | NFR |
| **Атрибут качества** | Reliability |
| **Приоритет** | P1 |
| **Статус** | УТВЕРЖДЕНО |
| **Источник** | — |
| **Критерии приёмки** | — |

---

## Что описывает

Определяет обязательный состав архивируемых артефактов, сроки хранения и правила передачи знаний при decommission.

## Обязательный состав архива

| Категория | Содержимое | Формат |
|---|---|---|
| Data export | TBox/ABox export, version history, audit logs | `.ttl`, `.jsonl`, `.git`, `.yaml` |
| Security evidence | access revocation logs, key rotation evidence, policy snapshots | `.json`, `.yaml` |
| Operational evidence | runbooks, incident timeline, migration/decommission reports | `.md`, `.pdf`, `.json` |
| Compliance evidence | approvals, customer confirmations, destruction certificates | `.pdf`, `.json` |

## Сроки хранения

| Артефакт | Retention |
|---|---|
| Export manifests + checksums | >= 3 года |
| Audit/security evidence | >= 3 года |
| Contractual/compliance evidence | >= 5 лет (или дольше по договору) |
| Migration runbooks и handover docs | >= 3 года |

## Процедура отзыва доступов

- Все сервисные и пользовательские привилегированные доступы должны быть отозваны <= 24 часов после decommission cutover.
- Оставшиеся активные токены после 24 часов: 0.
- Подтверждение отзыва фиксируется в `decommission-access-revocation-report.json`.

## Передача знаний

- Обязателен handover-пакет: архитектурная схема, операционные инструкции, карта зависимостей, known issues.
- Минимум 1 handover session с записью и протоколом для заказчика/эксплуатации.
- Подписание customer acceptance по передаче знаний: <= 10 рабочих дней после передачи.

## Контроль полноты

- Decommission считается завершенным только при статусе `archive_complete=true`.
- Отсутствие любой обязательной категории артефактов блокирует закрытие деcommission задачи.

## Archive Storage Environment (дополнение)

### Региональная и физическая изоляция

| Тип деплоя | Требование | Реализация |
|------------|------------|------------|
| **SaaS (VEDO managed)** | Архивы хранятся в отдельном AWS/Yandex регионе (не том же, где production) | S3 Cross-Region Replication (CRR) + WORM |
| **SaaS** | Доступ к архивам имеет отдельный service account (не production) | IAM roles, отдельные credentials |
| **On-premise** | Архивы должны храниться на отдельном физическом сервере или в отдельном ЦОД | Рекомендация (не блокирующее требование) |
| **Air-gapped** | Архивы хранятся на отдельном носителе (не на том же, где production) | Рекомендация |

### Минимальные требования к хранению

| Параметр | SaaS (VEDO managed) | On-premise (минимально) |
|----------|--------------------|------------------------|
| **Количество регионов** | 2 (активный + архивный) | 1 (рекомендовано 2) |
| **WORM (Object Lock)** | ✅ Обязателен | ⚠️ Рекомендован |
| **Отдельные credentials** | ✅ Обязательны | ⚠️ Рекомендованы |
| **Физическая изоляция** | N/A (облако) | Отдельный сервер / ЦОД |

### Проверка доступности архивов

| Проверка | Частота | Действие при недоступности |
|----------|---------|----------------------------|
| Запись тестового файла в архив (check) | Ежедневно | P1 алерт |
| Чтение тестового файла из архива (check) | Ежедневно | P1 алерт |
| Восстановление из архива (drill) | Ежеквартально | P0 алерт при ошибке |

### Пример конфигурации для SaaS (S3)

```yaml
# s3-archive-config.yaml
region: eu-central-1 (primary), eu-west-1 (archive)
bucket: vedo-archive-production
object_lock_enabled: true
object_lock_retention_days: 3650  # 10 лет
versioning_enabled: true
cross_region_replication:
  destination_bucket: arn:aws:s3:::vedo-archive-dr
  destination_region: eu-west-1
  metrics: enabled
iam_policy:
  - effect: "Allow"
    action:
      - "s3:PutObject"
      - "s3:GetObject"
    resource: "arn:aws:s3:::vedo-archive-production/*"
    condition:
      StringEquals:
        "s3:x-amz-object-lock-mode": "GOVERNANCE"
```

### Ответственность

| Роль | Ответственность |
|------|-----------------|
| **SRE** | Настройка S3 CRR, мониторинг доступности архивов |
| **Security Lead** | Проверка изоляции credentials |
| **Product Owner** | Утверждение выбора регионов (data residency) |

## Бизнес-правила

- Архивирование только в immutable/WORM хранилище.
- Редактирование архивных артефактов после фиксации запрещено.
- Удаление архива до истечения retention запрещено, кроме legal obligations с отдельным approval.
- Для SaaS архивы должны храниться как минимум в 2 регионах (активный + архивный).
- Для on-premise рекомендована физическая изоляция (отдельный сервер / ЦОД).
- Доступ к архивам через отдельные service account (не production credentials).
