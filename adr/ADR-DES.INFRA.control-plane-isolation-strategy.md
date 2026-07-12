# ADR-DES.INFRA.control-plane-isolation-strategy

**Дата:** 2026-05-17  
**Статус:** Принято

## Контекст

Сбои control plane в облачных системах приводят к длительной недоступности операций управления даже при частично живом data plane. Нужна формальная изоляция control plane и измеримый failover.

## Требование-источник

- [control-plane-isolation.md](requirements/REQ-NFR.INFRA.control-plane-isolation.md)

## Решение

Разделить control plane и data plane на уровне deployment, сетевых политик и секретов; ввести out-of-band break-glass контур и межрегиональный failover control plane с RTO <= 15 минут.

## Рассмотренные альтернативы

- Общий control/data plane — отклонено (общая точка отказа).
- Break-glass без отдельной auth-цепочки — отклонено (компрометация наследует blast radius).

## Последствия

- **Плюсы:** выше управляемость при авариях, предсказуемый recovery.
- **Минусы:** усложнение инфраструктуры и runbook.
- **Смягчение:** ежемесячные smoke-тесты и квартальные game day.

---
