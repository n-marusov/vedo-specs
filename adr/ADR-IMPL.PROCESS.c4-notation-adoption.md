# ADR-IMPL.PROCESS.c4-notation-adoption

**Дата:** 2026-05-09  
**Статус:** Принято

## Контекст

Необходимо стандартизировать документирование архитектуры.

## Требование-источник
- [architecture-documentation.md](requirements/REQ-FUN.PROCESS.architecture-documentation.md)

## Решение

Использовать C4 notation (Context, Container, Component, Code) для документирования архитектуры с рендерингом через Mermaid C4 диаграммы.

C4 обеспечивает единый иерархический формат архитектурных диаграмм для всех уровней детализации — от контекста системы до кода — и нативно рендерится в Markdown через Mermaid, что устраняет разрозненность форматов и упрощает ревью архитектурных изменений.

Хранить C4-диаграммы в `human/artifacts/c4-architecture.md`.

## Последствия

- Единый формат для всех архитектурных диаграмм
- Mermaid C4 упрощает рендеринг в markdown
- Диаграммы хранятся в `human/artifacts/c4-architecture.md`

---
