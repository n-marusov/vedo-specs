# ADR-DES.DATA.decommission-archive-retention-strategy

**Дата:** 2026-05-17  
**Статус:** Принято

## Контекст

Артефакты decommission хранились без единого обязательного состава и retention-политики. Это мешало аудиту, передаче знаний и последующему compliance review.

## Требование-источник

- [decommission-archive-retention.md](requirements/REQ-NFR.DATA.decommission-archive-retention.md)
- [decommission-export.md](requirements/REQ-NFR.DATA.decommission-export.md)
- [secure-erase.md](requirements/REQ-NFR.DATA.secure-erase.md)

## Решение

Установить обязательный состав decommission archive, сроки retention по категориям артефактов, SLA отзыва доступов и критерий завершения `archive_complete=true`.

## Рассмотренные альтернативы

- Архив только export-пакета — отклонено (нет security/compliance доказательств).
- Retention без immutable storage — отклонено (риск неаудируемых изменений).

## Последствия

- **Плюсы:** воспроизводимый audit trail и формализованная передача знаний.
- **Минусы:** рост требований к архивному хранилищу.
- **Смягчение:** tiered storage и классификация артефактов по срокам хранения.

---
