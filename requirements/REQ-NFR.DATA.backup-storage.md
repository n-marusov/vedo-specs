# Размещение резервных копий

| Атрибут | Значение |
|---------|----------|
| **ID** | REQ-NFR.DATA.backup-storage |
| **Уровень** | NFR |
| **Атрибут качества** | Reliability |
| **Приоритет** | P0 |
| **Статус** | УТВЕРЖДЕНО |
| **Источник** | — |
| **Критерии приёмки** | — |

---

## Обзор

VEDO Core использует комбинированную стратегию хранения backup по принципу 3-2-1: минимум три копии данных, минимум два типа хранилища, минимум одна копия вне основной площадки.

## Политика хранения

| Уровень | Назначение | Тип хранилища | Пример |
|---------|------------|---------------|--------|
| Локальное быстрое восстановление | Быстрое восстановление после типовых ошибок | Локальный volume / backup node | SSD/HDD рядом с production |
| Основное backup storage | Основная резервная копия | S3/MinIO-compatible object storage | S3, MinIO, корпоративный object storage |
| Off-site / cold copy | Disaster recovery при потере площадки | Географически отдельное cold storage | Другой DC, Glacier-like storage, offline vault |

## Модели развёртывания

| Модель deployment | Требование к backup storage |
|------------------|----------------------------|
| SaaS | Управляемое local restore + managed object storage + off-site cold copy |
| Private Cloud | Одобренное заказчиком object storage + off-site copy внутри разрешенной зоны |
| On-premise | Управляемый заказчиком local backup + customer object storage/NAS + off-site по политике заказчика |
| Air-gapped | Локальное backup storage + offline/off-site носитель без зависимости от внешнего интернета |

## Обоснование

Только локальные backup не покрывают потерю площадки, ransomware и ошибки администратора. Только облачные backup недостаточны для on-premise и air-gapped сценариев. Поэтому базовая production policy должна быть комбинированной и адаптируемой под deployment model.

## Минимальные требования

- Backup storage должен быть отделен от primary datastore credentials.
- Off-site copy должна переживать потерю основной площадки.
- Для object storage должно быть включено versioning или эквивалентная защита от перезаписи.
- Для cold storage должен быть определен restore procedure, иначе copy не считается валидным backup.
- Для air-gapped deployment не должно быть обязательной зависимости от public cloud.

## Immutable backup — защита от компрометации

Минимальное требование (обязательно для всех deployment model):

| Характеристика | Требование |
|----------------|------------|
| **Изоляция учётных данных** | Credentials для backup-bucket хранятся отдельно от production-credentials. Доступ production-кластера к backup-bucket — write-only (без права на удаление). |
| **Региональная изоляция** | Backup-bucket находится в другом регионе, чем production-кластер. |
| **Защита от удаления** | Object Lock (WORM) или Bucket Versioning с защитой от удаления версий. |
| **Отдельная административная зона** | Управление backup-bucket — только из отдельной administrative сети или dedicated management plane. |

Рекомендуемое усиление (опционально):

- Полная изоляция аккаунтов: отдельный облачный аккаунт/проект (AWS Organization, отдельный GCP project).
- Cross-cloud копия: одна копия у другого облачного провайдера.

Обоснование (урок Code Spaces 2014): злоумышленник, получив доступ к production-аккаунту, смог удалить все backup, потому что они хранились в том же аккаунте с теми же правами. Отдельный bucket с write-only доступом и Object Lock предотвращает этот сценарий.

Что не допускается:

- Единственная копия backup в том же S3-bucket, что и production-данные.
- Одни и те же IAM-credentials для production-кластера и управления backup.
- Отсутствие Object Lock или эквивалента на backup-bucket.
- Backup в том же регионе, что и production, без дополнительной копии в другом регионе.

### Закрывает

- `4.3 Аварии/Катастрофы × Эксплуатация / Обслуживание`

## Приоритет восстановления

При восстановлении система использует источники в порядке скорости и надежности:

1. Local fast restore copy.
2. Primary object storage backup.
3. Off-site / cold copy.

## Не цели

- VEDO Core не навязывает конкретного cloud vendor.
- On-premise заказчик может использовать собственный backup perimeter, если он сохраняет свойства 3-2-1.
- Off-site copy не обязана быть online, если RTO/RPO для соответствующего deployment model остаются достижимыми.
