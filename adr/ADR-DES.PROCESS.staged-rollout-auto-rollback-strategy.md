# ADR-DES.PROCESS.staged-rollout-auto-rollback-strategy

**Дата:** 2026-05-17  
**Статус:** Принято

## Контекст

Инциденты крупных SaaS-платформ показывают, что быстрый rollout без формальных порогов блокировки и auto-rollback приводит к массовой деградации. В текущих артефактах стратегия rollout была описана, но часть формулировок оставалась рекомендательной.

## Требование-источник

- [rollout-safety-gates.md](requirements/REQ-NFR.PROCESS.rollout-safety-gates.md)
- [deployment-strategy.md](requirements/REQ-CON.INFRA.deployment-strategy.md)

## Решение

Принять обязательную staged rollout policy для production: `0% -> 5% -> 25% -> 50% -> 100%` с hard-threshold блокировкой перехода и автоматическим rollback при нарушении порогов качества.

## Рассмотренные альтернативы

- Manual rollout без hard-threshold — отклонено (высокий риск человеческой ошибки).
- Instant 100% rollout — отклонено (неприемлемый blast radius).
- Staged rollout + manual rollback — отклонено (слишком большой MTTR).

## Последствия

- **Плюсы:** снижение blast radius, ускорение восстановления, воспроизводимый release gate.
- **Минусы:** более длинный pipeline, рост требований к observability.
- **Смягчение:** параллелизация проверок и стандартизация evidence.

---
