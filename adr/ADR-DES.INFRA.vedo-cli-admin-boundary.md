# ADR-DES.INFRA.vedo-cli-admin-boundary

**Дата:** 2026-05-12  
**Статус:** Принято  
**Обновлено:** 2026-05-19 (добавлено управление тикетами через `vedo-cli`)

## Контекст

Администрирование VEDO Core включает backup/restore, миграции Neo4j/PostgreSQL, air-gapped подготовку, экспорт/импорт онтологий, диагностику инцидентов и управление тикетами поддержки. Без единой административной утилиты эти операции распадаются на `kubectl`, `helm`, `neo4j-admin`, `psql`, S3-клиенты и ручные Grafana/Loki/Tempo/Prometheus запросы.

Разрозненное администрирование повышает риск ошибки, усложняет аудит privileged actions и плохо подходит для air-gapped environments.

## Требование-источник
- [cli-admin-tool.md](requirements/REQ-USR.INFRA.cli-admin-tool.md)
- [vedo-cli-specification.md](requirements/REQ-FUN.INFRA.vedo-cli-specification.md)
- [ticket-management-system.md](requirements/REQ-FUN.INTEGRATION.ticket-management.md)

## Решение

Зафиксировать `vedo-cli` как единую административную утилиту командной строки для экосистемы VEDO Core, ответственную за backup/restore, миграции, air-gap подготовку, операции с онтологиями, диагностику и управление тикетами поддержки.

Единый интерфейс для DevOps, инженеров поддержки и администраторов онтологий устраняет разрозненность `kubectl`, `helm`, `neo4j-admin`, `psql` и S3-клиентов. Privileged операции становятся воспроизводимыми и аудируемыми через единый security/audit контур. Air-gapped подготовка получает формальный интерфейс, а backup, migration и diagnostics покрываются acceptance tests и runbooks.

Ограничить зону ответственности `vedo-cli` операциями, требующими автоматизации или повышенных привилегий, не заменяя web UI и public API. Все действия логировать через единый security/audit контур. Destructive-команды снабдить guardrails и MFA, прямой production-доступ к БД вне `vedo-cli` допускать только как согласованный break-glass процесс.

`vedo-cli` также обеспечивает управление тикетами через единый backend Ticket Management (тот же жизненный цикл, что и в UI):
- `vedo-cli ticket create --title "<title>" --description "<text>" --category <category> --severity <level>`
- `vedo-cli ticket list --status <status> --category <category> --source <manual|telemetry>`
- `vedo-cli ticket get --id <ticket_id>`
- `vedo-cli ticket update --id <ticket_id> --priority <p0|p1|p2|p3> --assignee <user>`
- `vedo-cli ticket comment --id <ticket_id> --text "<comment>"`
- `vedo-cli ticket close --id <ticket_id> --resolution "<text>"`
- `vedo-cli ticket reopen --id <ticket_id> --reason "<text>"`
- `vedo-cli ticket delete --id <ticket_id>` (guardrail G3: MFA + env guard + typed confirmation)

Операции `vedo-cli ticket` обязаны:
- работать с тем же источником истины, что и web UI;
- фиксировать канал выполнения (`channel=cli`) в истории тикета;
- писать расширенный аудит (actor, role, command, ticket_id, trace_id/correlation_id, result, timestamp).

`vedo-cli` также обеспечивает управление Merge Request'ами:
- `vedo-cli mr create --source feature/xyz --target main`
- `vedo-cli mr list --status open`
- `vedo-cli mr review --id 123 --approve`
- `vedo-cli mr merge --id 123`

## Рассмотренные альтернативы

| Альтернатива | Причина отклонения |
|--------------|--------------------|
| Только web UI для администрирования | Не подходит для automation, CI/CD, SSH-only и air-gapped сценариев |
| Набор shell scripts | Сложно версионировать API команд, нет единого audit trail, выше риск drift между окружениями |
| Использовать только native tools (`kubectl`, `psql`, `neo4j-admin`) | Операции становятся зависящими от поставщика/инструмента и не выражают доменные инварианты VEDO Core |
| Дать администраторам прямой доступ к БД | Повышает риск обхода access-control, потери данных и неполного аудита |

## Последствия

**Положительные последствия:**
- Единый интерфейс для DevOps, инженеров поддержки и администраторов онтологий.
- Privileged operations становятся воспроизводимыми и аудируемыми.
- Air-gapped подготовка и проверка получают формальный интерфейс.
- Backup, migration и diagnostics требования могут быть покрыты acceptance tests и runbooks.
- Поддержка получает CLI-управление тикетами без расхождения с UI-жизненным циклом.

**Отрицательные последствия:**
- `vedo-cli` становится критичным компонентом delivery и support lifecycle.
- Нужно поддерживать совместимость CLI-команд с версиями серверных служб и диаграмм развёртывания.
- Требуется отдельная документация команд, exit codes и machine-readable output.
- Расширяется поверхность ошибок при управлении тикетами (конкуренция UI/CLI, конфликты обновлений).

**Меры снижения рисков:**
- Версионировать `vedo-cli` вместе с VEDO Core release.
- Поддерживать JSON output для automation и человекочитаемый output для интерактивной работы.
- Покрыть проверками ключевые команды: backup verify, restore drill, migrate rollback, air-gap package check, diagnose trace.
- Ввести идемпотентность и optimistic locking для `ticket update/close/reopen/delete`.
- Покрыть контрактными тестами сценарии UI+CLI конкурентных изменений тикета.

---
