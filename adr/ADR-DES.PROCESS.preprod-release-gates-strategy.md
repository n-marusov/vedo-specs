# ADR-DES.PROCESS.preprod-release-gates-strategy

**Дата:** 2026-05-17  
**Статус:** Принято

## Контекст

Для части релизов не было единого блокирующего pre-prod gate, явно ограничивающего переход к 100% production-трафика. Это повышает риск выката дефекта в полном масштабе.

## Требование-источник

- [preprod-release-gates.md](requirements/REQ-FUN.PROCESS.preprod-release-gates.md)
- [rollout-safety-gates.md](requirements/REQ-NFR.PROCESS.rollout-safety-gates.md)

## Решение

Ввести обязательные pre-prod категории проверок (smoke, security, performance, data integrity, compatibility) с hard-block порогами и запретом 100% rollout без `PASS`.

## Рассмотренные альтернативы

- Частичный pre-prod набор для minor релизов — отклонено (пропуск regressions в API/security).
- Manual sign-off без машинного gate — отклонено (невоспроизводимость и человеческий фактор).

## Последствия

- **Плюсы:** снижение вероятности массового инцидента при полном rollout.
- **Минусы:** увеличение времени pre-release pipeline.
- **Смягчение:** параллельный запуск тестовых категорий и оптимизация smoke suites.

---
