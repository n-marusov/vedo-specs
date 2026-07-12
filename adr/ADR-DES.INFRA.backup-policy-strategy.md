# ADR-DES.INFRA.backup-policy-strategy

**Дата:** 2026-05-10  
**Статус:** Принято

## Контекст

VEDO Core хранит данные разных типов с разной ценностью, размером и частотой изменения: TBox, ABox, Version Store, LFS-объекты и аудиторские логи. Единый backup retention или единый формат backup не покрывает одновременно быстрый restore, auditability, переносимость, стоимость хранения и корпоративные и compliance требования.

## Требование-источник
- [backup-retention.md](requirements/REQ-NFR.DATA.backup-retention.md)
- [backup-format.md](requirements/REQ-NFR.DATA.backup-format.md)
- [backup-storage.md](requirements/REQ-NFR.DATA.backup-storage.md)
- [runbook-ownership-drill-evidence.md](requirements/REQ-NFR.INFRA.runbook-ownership-drill-evidence.md)

## Решение

Принять многоуровневую backup policy с раздельным retention, форматами и storage topology (3-2-1 стратегия) по типам данных.

Раздельный retention (TBox 90 дней active + 1 год archive, Version Store 90 дней + 7 лет, audit 1 год + 3 года) соответствует разной критичности и размеру данных. Canonical Turtle для TBox сохраняет человекочитаемость и переносимость. 3-2-1 стратегия (local fast restore copy, S3/MinIO primary backup, off-site/cold copy) защищает от потери площадки, ransomware и ошибок администратора. On-premise и air-gapped развёртывания не зависят от public cloud.

TBox бэкапить в canonical Turtle (`.ttl.gz`), ABox — graph store dump + WAL, Version Store — `pg_dump -Fc` + PostgreSQL WAL, LFS — raw objects в S3/MinIO, аудиторские выгрузки — structured export для инспекции. Backup storage отделить от primary datastore credentials, включить object versioning. Для on-premise допустить управляемый заказчиком периметр при сохранении свойств 3-2-1.

## Рассмотренные альтернативы

| Альтернатива | Причина отклонения |
|--------------|--------------------|
| Единый retention 30 дней для всех данных | Недостаточно для Version Store и аудиторских логов, избыточно для LFS |
| Только full snapshots | Простой restore, но высокая стоимость хранения и слабый PITR |
| Только incremental/WAL | Экономит место, но повышает риск сложного или невозможного restore без full snapshot |
| Только canonical Turtle | Подходит для TBox, но непригоден как единственный production backup для ABox и Version Store |
| Только cloud backup | Не покрывает on-premise и air-gapped развёртывания |
| Только local backup | Не покрывает потерю площадки, ransomware и ошибки администратора |

## Последствия

**Положительные последствия:**
- Backup policy соответствует разной критичности и размеру TBox, ABox, Version Store, LFS и логов.
- TBox остаётся человекочитаемым и переносимым через canonical Turtle.
- ABox и Version Store получают production-grade restore через full dumps + WAL/PITR.
- 3-2-1 storage снижает риск потери данных при аварии площадки или ransomware.
- On-premise и air-gapped развёртывания не зависят от public cloud.

**Отрицательные последствия:**
- Эксплуатация backup становится сложнее: несколько форматов, retention windows и storage tiers.
- Стоимость хранения выше из-за архивного Version Store на 7 лет и audit logs на 3 года.
- Restore testing должен покрывать несколько сценариев и типов данных.

**Меры снижения рисков:**
- Backup policy формализуется в `BACKUP-001` и проверяется через `backup-policy` test spec.
- Для on-premise допускается управляемый заказчиком периметр при сохранении свойств 3-2-1.
- Бессрочное хранение допускается только как внешний архив/архив заказчика; для логов бессрочное хранение запрещено.

---
