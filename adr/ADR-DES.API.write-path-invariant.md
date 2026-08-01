# ADR-DES.API.write-path-invariant

**Дата:** 2026-08-01  
**Статус:** Принято

## Контекст

Текущий код нарушает инвариант пути записи: ontology-service выполняет прямые CRUD-записи в Neo4j (классы, свойства, индивиды) без ветки и коммита; импорт выполняется напрямую (replace/merge). Versioning и CRUD разъединены: коммит пишет дельту в PostgreSQL и НЕ применяет её к Neo4j — только checkout пересоздаёт состояние и обновляет внутренний state.

Следствия текущей модели:
- Нет аудиторского следа для каждой мутации (кто, что, когда, ветка).
- Невозможен откат к произвольной точке (Git-парадигма не работает на уровне данных).
- Риск повреждения данных при конкурентных записях.
- Невозможно ввести внешних AI-агентов без жёсткого контейнера записи.

## Требование-источник

- [REQ-CON.SECURITY.write-path-invariant.md](../requirements/REQ-CON.SECURITY.write-path-invariant.md)
- [ADR-DES.DATA.storage-stack-strategy.md](ADR-DES.DATA.storage-stack-strategy.md) — «deltas per commit»
- [ADR-DES.PROCESS.merge-request-strategy.md](ADR-DES.PROCESS.merge-request-strategy.md) — MR-жизненный цикл
- [REQ-FUN.INTEGRATION.collaboration.md](../requirements/REQ-FUN.INTEGRATION.collaboration.md) — §3.3

## Решение

**Принять write-path invariant как первоклассное архитектурное ограничение: НЕ существует пути мутации состояния онтологии вне конвейера версионирования.**

Каждая мутация:

```
branch → commit → MR → review → merge
```

- **REST write-эндпоинты** работают с branch-local снимками (`branch snapshots`), никогда с живой онтологией.
- **Внутренние обработчики** вызывают versioning-service; они никогда не пишут в хранилище напрямую.
- **Импорт/экспорт** становятся операциями, порождающими коммиты.
- **Каждая мутация** = залогированный коммит с аудиторским следом (кто, что, когда, ветка).

### Контейнер внешних AI-агентов (subsection)

- AI-агенты интегрируются через платформенный API (F6), а не как внутренние акторы.
- **Capability scopes** (не RBAC role ladder): `read`, `propose:abox`, `propose:tbox`, `mr:create`, `publish:restricted`.
- **Запрещённые scopes:** `write:direct`, `merge:self`, `publish:public` (требует человека).
- **Autonomy × Authority:** ABox auto-propose допустим; TBox требует детерминированный gate; public-публикация = человеческий gate.
- **Security containment:** AI пишет ТОЛЬКО в собственные ветки в пространстве `branches/{client}/*`.
- **Identity separation:** proposer ≠ approver ≠ publisher даже при полной автоматизации.
- **Review gate** = детерминированный код (reasoner, policy engine); никогда «AI ревьюит AI».

## Рассмотренные альтернативы

| Альтернатива | Причина отклонения |
|--------------|--------------------|
| A) Текущая модель (прямые записи + разъединённый versioning) | Нет аудиторского следа, нет инварианта, риск повреждения данных |
| B) Двухфазная запись (прямая запись + отложенный commit) | Гонки, сложность, двойная семантика |
| C) Полный инвариант (выбрано) | Чистейшая модель, Git-парадигма корректна, единый путь мутации |

## Последствия

**Положительные:**
- Полный аудиторский след каждой мутации.
- Git-парадигма отката работает на уровне данных.
- Внешние AI-агенты получают безопасный контейнер (capability scopes + namespace containment).
- Единый путь мутации упрощает тестирование и reasoning.

**Отрицательные:**
- M10 MR workflow становится обязательным enforcement backbone — без него инвариант не выполним.
- ontology-service нуждается в рефакторинге: все write-пути → versioning-service.
- Импорт/экспорт становятся commit-порождающими операциями.
- Существующие direct-write тесты требуют миграции.

**Меры снижения рисков:**
- Code guardrails (deprecation headers, planned stubs) применяются в M5 до полной миграции.
- Тесты, упражняющие прямые записи, аннотируются `// NOTE: violates ADR-DES.API.write-path-invariant; to be migrated in M10`.
- Планирование M10 включает enforcement backbone как критический путь.

## Связанные ADR

- [ADR-DES.DATA.storage-stack-strategy.md](ADR-DES.DATA.storage-stack-strategy.md) — «deltas per commit»
- [ADR-DES.PROCESS.merge-request-strategy.md](ADR-DES.PROCESS.merge-request-strategy.md) — MR-жизненный цикл (enforcement backbone)
- [ADR-DES.API.rest-gitlab-alignment.md](ADR-DES.API.rest-gitlab-alignment.md) — REST-структура (branch-local snapshots)
- [REQ-CON.SECURITY.write-path-invariant.md](../requirements/REQ-CON.SECURITY.write-path-invariant.md)

---
