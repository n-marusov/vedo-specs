# ADR-DES.SECURITY.destructive-command-guardrails

**Дата:** 2026-05-13  
**Статус:** Принято

## Контекст

P0 runbook и CLI должны предотвращать разрушительные ошибки оператора. Общий флаг `--confirm-production-risk` не различает степень опасности операций и десенситизирует администратора.

## Требование-источник
- [rollback-runbook.md](requirements/REQ-NFR.DATA.rollback-runbook.md)
- [migration-runbook.md](requirements/REQ-NFR.DATA.migration-runbook.md)

## Решение

Ввести закрытый перечень destructive-команд D1–D10 с четырьмя уровнями guardrails G1–G4: G1 — typed confirmation, G2 — environment guard + typed confirmation, G3 — MFA или second admin confirmation, G4 — cool-down + verification.

Градация guardrails предотвращает десенситизацию администратора от единого флага --confirm-production-risk: каждая destructive-команда получает ровно тот уровень защиты, который соответствует её риску — от typed confirmation для branch delete до cool-down с pre-flight verification для tenant delete. Environment guard для G2+ требует точного ввода имени окружения при production и проверяет approved management network.

G1: typed confirmation (branch delete, commit revert); G2: environment guard (tenant seed --force); G3: MFA/second admin (ontology delete, backup delete, restore, migrate rollback); G4: cool-down + verification (tenant delete, decommission --purge-data).

## Рассмотренные альтернативы

| Альтернатива | Причина отклонения |
|--------------|--------------------|
| Один общий `--confirm-production-risk` | Не учитывает разные уровни опасности и создаёт false confidence |
| Только документационный запрет в runbook | Не предотвращает ошибочную команду на уровне инструмента |
| Полный запрет destructive-команд | Невозможно: нужны decommission, restore, rollback и lifecycle operations |

## Последствия

**Положительные последствия:**
- GitLab DB incident class mitigated: CLI различает staging и production.
- Самые опасные операции получают cool-down и pre-flight verification.
- Guardrails становятся проверяемыми через CLI contract tests.

**Отрицательные последствия:**
- Некоторые операции становятся медленнее и требуют второго подтверждения.
- Нужно поддерживать актуальный реестр destructive-команд.

**Меры снижения рисков:**
- Реестр D1–D10 хранится как часть runbook/security artifacts.
- Новые destructive-команды не допускаются без явного уровня G1–G4.

---
