# ADR-DES.UI.import-export-safety-strategy

**Дата:** 2026-05-10  
**Статус:** Принято

## Контекст

Импорт/экспорт онтологий является миграционным мостом между VEDO Core и существующими инструментами, включая Protege. Частичный импорт без отчёта, импорт без preview или экспорт, который нельзя открыть в Protege, являются критичными UX и compatibility проблемами.

## Требование-источник
- [import-export.md](requirements/REQ-USR.UI.import-export.md)

## Решение

Разделить import/export конвейер на четыре стадии: parse → dry-run plan → apply as versioned batch → report artifact.

Pipeline гарантирует безопасный preview без изменения данных до подтверждения, пакетное применение с созданием versioned commit для rollback и аудита, детальный Import Report по каждой сущности и canonical serialisation с совместимостью с Protege — пользователь видит последствия импорта до его применения и получает диагностику после.

Import API строит Import Plan до мутации данных с detected conflicts и estimated changes; apply выполняется пакетно через Versioning Service; Export использует canonical serialization конвейер и compatibility test profile.

## Рассмотренные альтернативы

| Альтернатива | Причина отклонения |
|--------------|--------------------|
| Parse-and-apply в одном запросе | Нет безопасного preview и сложнее rollback |
| Отчёт только в UI state | Диагностика теряется после reload и недоступна для поддержки |
| Export без canonical profile | Возвращаются нестабильные diff и риск несовместимости с Protege |

## Последствия

**Положительные последствия:**
- Import/export становится воспроизводимым конвейером, а не UI-операцией.
- Plan/Report можно тестировать и хранить для аудита.
- Versioning Service получает точку привязки массовых изменений.

**Отрицательные последствия:**
- Больше сложность серверной части: plan schema, report storage, batch apply.
- Dry-run должен быть достаточно точным, чтобы не расходиться с apply.

**Меры снижения рисков:**
- Версионировать Import Plan/Report schema.
- Начать с Turtle/RDF/XML/OWL MVP profile и расширять совместимость итеративно.

---
