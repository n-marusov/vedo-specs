# ADR-DES.SECURITY.authorization-policy-gates-strategy

**Дата:** 2026-05-17  
**Статус:** Принято

## Контекст

Регрессии авторизации (BOLA/BFLA/tenant isolation) могут приводить к критическим утечкам. Нужен жесткий release gate с policy-as-code.

## Требование-источник

- [authorization-regression-gates.md](requirements/REQ-NFR.SECURITY.authorization-regression-gates.md)

## Решение

Сделать authorization regression gates обязательными и блокирующими для production release: 0 критических дефектов, 100% покрытие обязательных негативных тестов, подпись policy bundles.

## Рассмотренные альтернативы

- Неформальный security review без gate — отклонено (неизмеримо).
- Выборочное тестирование только топ-эндпоинтов — отклонено (пропуск cross-tenant путей).

## Последствия

- **Плюсы:** предотвращение критических утечек до выкладки.
- **Минусы:** увеличение времени pre-release проверки.
- **Смягчение:** оптимизация тестового набора и ограничение длительности gate до 15 минут.

---
