# Требования к удобству runbook

| Атрибут | Значение |
|---------|----------|
| **ID** | REQ-NFR.INFRA.runbook-usability |
| **Уровень** | NFR |
| **Атрибут качества** | Usability |
| **Приоритет** | P1 |
| **Статус** | УТВЕРЖДЕНО |
| **Источник** | — |
| **Критерии приёмки** | — |

---

## Целевые характеристики

| Характеристика | Требование | Способ проверки |
|---|---|---|
| **Целевой оператор** | Operator с базовой подготовкой (не разработчик сервиса) | Staging drill с новым инженером поддержки |
| **Целевое время исполнения P0 runbook** | ≤ 30 минут без escalation к разработчикам | Замер времени в drill-сценарии |
| **Самостоятельность** | Runbook НЕ требует: чтения кода сервисов, знания внутренней топологии БД, ручного поиска в Grafana/Loki/Tempo дашбордах | Drill observer checklist |
| **Единая точка входа в диагностику** | `vedo-cli diagnose trace --id <trace_id>` | Выполняется первой командой runbook |
| **Единая точка входа в восстановление** | `vedo-cli restore`, `vedo-cli migrate rollback`, `vedo-cli ontology restore` — в зависимости от сценария | Покрытие P0 recovery-путей |
| **Формат runbook** | `.md` файл в репозитории, доступный offline (Antora build) | Проверяется link checker и air-gap package verify |

## Drill-сценарий (пример)

**Сценарий:** потеря connectivity Neo4j primary в staging в рабочее время.

1. Alert → ticket создан автоматически, escalation timer запущен.
2. Оператор открывает runbook `p0-neo4j-connectivity-loss.md`.
3. Выполняет `vedo-cli diagnose trace --id <trace_id>` → видит `neo4j-cluster`, `leader-unavailable`, рекомендацию проверить `neo4j-core-primary`.
4. Выполняет описанные команды проверки `neo4j-admin`, видит состояние пода, принимает решение: `vedo-cli restore` или рестарт пода по инструкции.
5. Восстановление подтверждено smoke-тестом runbook.
6. **Время от alert до restore verified ≤ 30 минут, без эскалации на L2/L3.**

## Соответствие ADR и артефактам

| Артефакт | Как отражено |
|---|---|
| `ADR-DES.INFRA.vedo-cli-diagnostics-entrypoint` | `vedo-cli diagnose` как единая точка входа в диагностику, P0 ≤ 10 минут |
| `ADR-DES.INFRA.vedo-cli-admin-boundary` | `vedo-cli restore`, `vedo-cli migrate rollback` как единые точки восстановления |
| `ADR-DES.OPS.support-sla-and-escalation-strategy` | Матрица эскалации, таймеры, drill-обязательства |
| `ADR-DES.INFRA.airgap-offline-deployment-strategy` | Runbook должен быть доступен offline |
| `ADR-IMPL.STACK.antora-docs-adoption` | Runbook собирается Antora, проверяется link checker |
| `human/artifacts/requirements/REQ-NFR.DOC.documentation-training.md` | Incident Diagnosis Guide как обязательный документ MVP |

## Неприемлемые практики

- Runbook, который начинается с «открой Grafana и найди dashboard…» и требует ручной навигации — **неприемлем** для P0.
- Runbook, который в середине говорит «спроси у разработчика, почему так» — **неприемлем** без явной эскалационной границы.
- Runbook, который ссылается на online-ресурсы без offline-копии — **неприемлем** для air-gapped deployments.

## Destructive-command guardrails

### Уровни защиты

| Уровень | Название | Описание |
|---------|----------|----------|
| **G1** | Typed confirmation | Ввод имени удаляемого объекта (tenant, ontology, account) |
| **G2** | Environment guard + typed confirmation | G1 + проверка, что команда НЕ в production |
| **G3** | Multi-factor confirmation | G2 + второй фактор или подтверждение вторым администратором |
| **G4** | Cool-down delay + full verification | G3 + пауза + проверка backup |

### Перечень destructive-команд

| ID | Команда | Область | Уровень | Механизм |
|----|---------|---------|---------|----------|
| D1 | `vedo-cli tenant delete` | Удаление tenant со всеми данными | G4 | Preview → typed confirmation → env guard → 24h cool-down → backup verify → purge |
| D2 | `vedo-cli ontology delete` | Удаление онтологии (TBox + ABox) | G3 | Preview → typed confirmation → env guard → verify no external dependencies |
| D3 | `vedo-cli account close` | Закрытие аккаунта пользователя | G3 | Preview → typed confirmation → owner transfer check → 30d cool-down |
| D4 | `vedo-cli decommission --purge-data` | Полное удаление данных при decommission | G4 | Verify export status `ready` → typed confirmation → env guard → 1h cool-down |
| D5 | `vedo-cli restore` (production) | Восстановление из backup с перезаписью | G3 | Env guard → backup verify → typed confirmation → read-only mode |
| D6 | `vedo-cli migrate rollback` | Откат миграции со сбросом данных | G3 | Env guard → verify pre-migration backup → typed confirmation → maintenance window |
| D7 | `vedo-cli backup delete` | Удаление резервной копии | G3 | List backups → typed confirmation (без cool-down, необратимо) |
| D8 | `vedo-cli tenant seed --force` | Перезапись данных tenant профилем | G2 | Env guard → typed confirmation |
| D9 | `vedo-cli branch delete` | Удаление ветки версионирования | G1 | Typed confirmation (без env guard, допустимо в dev) |
| D10 | `vedo-cli commit revert` | Откат коммита | G1 | Preview diff → typed confirmation |

### Поведение environment guard

При выполнении команды уровня G2+ на окружении, маркированном как `production`:

```
WARNING: Production environment detected.
This is a destructive operation that may result in data loss.
To proceed, type the full environment name: production
> _
```

Команда выполняется только после точного ввода имени окружения. Для SaaS — дополнительная проверка, что команда инициирована из approved management network.

### Обоснование

- Разные destructive-команды имеют разную степень опасности и требуют разных уровней защиты.
- GitLab DB incident (2017): guardrails не различали окружения — индивидуальный env guard для D4-D7 предотвращает повторение.
- Общий маркер `--confirm-production-risk` десенситизирует администратора и создаёт false confidence.
