# ADR-DES.INFRA.recovery-objectives-mandate

**Дата:** 2025-03-15  
**Статус:** Принято

## Контекст

VEDO Core работает с критичными данными онтологий (TBox — классы, свойства) и данными индивидов (ABox). Потеря данных или простой влияют на бизнес напрямую.

Из бизнес-требований: критический отказ — 4 часа, RPO TBox — 5 минут, RPO ABox — 15 минут.

## Требование-источник
- [sla-recovery.md](requirements/REQ-NFR.INFRA.sla-recovery.md)
- [deployment-tiers-reference-architecture.md](requirements/REQ-CON.INFRA.deployment-tiers.md)
- [runbook-ownership-drill-evidence.md](requirements/REQ-NFR.INFRA.runbook-ownership-drill-evidence.md)

## Решение

Принять целевые и предельные RTO по уровням критичности и RPO по типам данных: TBox 5 минут, ABox 15 минут, Version Store 1 час, LFS-объекты 24 часа.

Дифференцированные RTO и RPO соответствуют разной критичности данных: схема онтологии (TBox) восстанавливается быстрее всего, LFS-объекты допускают более длительное окно потери. Критический отказ с RTO 4 часа задаёт измеримую цель для инфраструктурных решений. RPO 5 минут для TBox через синхронную репликацию минимизирует потерю схемных знаний.

Для production-профиля реализовать синхронную репликацию graph store для TBox, асинхронную репликацию + WAL для ABox, ежечасный backup PostgreSQL для Version Store и geo-репликацию S3/MinIO для LFS. Neo4j Enterprise cluster использовать с BYOL/лицензией заказчика.

**Уточнение по deployment tiers (v1.0):**
- `Standard`: целевой `RTO=1 час`, `RPO=1 час`.
- `High`: целевой `RTO=15 минут`, `RPO=15 минут`.
- `Premium`: целевой `RTO=5 минут`, `RPO=5 минут`; для TBox — синхронная репликация, для ABox — асинхронная, с обязательной проверкой lag и регулярными DR-drills.
- Гарантии RTO/RPO валидны только при соответствии минимальной reference architecture выбранного tier.

## Последствия

- **Положительные последствия:** Клиенты получают гарантии, детальная процедура восстановления
- **Отрицательные последствия:** Стоимость инфраструктуры выше; Neo4j Enterprise cluster требует отдельной коммерческой лицензии или managed subscription
- **Смягчение:** MIT baseline не требует Neo4j Enterprise-only features; production HA profile использует BYOL/лицензию заказчика или альтернативное совместимое графовое хранилище

---
