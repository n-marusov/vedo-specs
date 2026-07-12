# ADR-DES.UI.wcag-accessibility-strategy

**Дата:** 2026-05-17  
**Статус:** Принято  
**Supersedes:** предыдущая версия от 2025-03-15

## Контекст

VEDO Core — редактор онтологий для инженеров знаний, ИТ-архитекторов. Основные пользователи не имеют физических ограничений, но:
- Госсектор и образование требуют WCAG 2.1 AA по ФЗ № 419-ФЗ
- Академические пользователи могут полагаться на скрин ридеры
- Игнорирование accessibility отрезает от госзакупок
- Enterprise и госзаказчики считают WCAG A на MVP недостаточным для участия в тендерах

## Требование-источник
- [accessibility.md](requirements/REQ-NFR.UI.accessibility.md)
- [wcag-mvp-scenarios.md](human/artifacts/wcag-mvp-scenarios.md) (детальная матрица сценариев)
- [accessibility.md](human/artifacts/accessibility.md) (уточнённый)

## Решение

Принять **WCAG 2.1 Level AA как целевой уровень для MVP** (обновление с Level A). Пять категорий критических сценариев обязаны соответствовать Level AA на момент релиза MVP:

| Категория | Сценарии | Критерии WCAG 2.1 AA |
|-----------|----------|----------------------|
| **A — Аутентификация** | Login, OAuth redirect, logout, session expiry | 2.4.7 Focus Visible, 3.3.1 Error Identification, 3.3.2 Labels or Instructions, 1.1.1 Non-text Content |
| **B — Навигация** | Sidebar, breadcrumbs, search, tree navigation, keyboard tab order | 2.4.3 Focus Order, 2.4.6 Headings and Labels, 2.4.7 Focus Visible, 1.3.1 Info and Relationships |
| **C — Редактирование** | ClassTree CRUD, PropertyPanel, form validation, inline edit | 3.3.2 Labels or Instructions, 3.3.1 Error Identification, 2.1.1 Keyboard, 1.4.3 Contrast (Minimum) |
| **D — Версионирование** | Branch switch, commit dialog, diff view, merge request | 2.1.1 Keyboard, 2.4.3 Focus Order, 1.4.3 Contrast (Minimum), 4.1.2 Name, Role, Value |
| **F — Публичный просмотр** | Public Ontology View, read-only graph navigation, search | 1.1.1 Non-text Content, 2.4.3 Focus Order, 1.4.3 Contrast (Minimum), 2.4.6 Headings and Labels |

**Исключения** (не требуют Level AA на MVP):

| Исключение | Обоснование |
|------------|-------------|
| 3D-граф (Canvas-based) | Природа 3D-визуализации не позволяет обеспечить AA без радикального перепроектирования. Компенсация: табличный fallback с keyboard-доступом |
| Drag-and-drop (ClassTree reorder) | Native D&D недоступен для скрин ридеров. Компенсация: альтернативный button-based интерфейс (Move Up / Move Down / Move to Parent) |
| SPARQL редактор (Syntax Highlight) | Сложность подсветки синтаксиса для скрин ридеров. Компенсация: plain-text textarea с label и error validation |

**PASS/FAIL критерий:** пользователь выполняет сценарий без мыши, без подсказок, за время ≤ 3× времени обычного пользователя.

**Инструменты валидации:**
- **Automated:** axe-core в CI (block critical/serious violations, fail pipeline)
- **Keyboard-only:** Playwright keyboard‑only тесты для каждого критического сценария
- **Manual:** NVDA/VoiceOver manual тесты перед каждым релизом MVP

## Рассмотренные альтернативы

| Альтернатива | Причина отклонения |
|--------------|---------------------|
| Level AAA | Требует радикального пересмотра UI, нецелесообразно |
| Только 3D-граф | Canvas недоступен для скрин ридеров, нет альтернативы |
| Без accessibility | Потеря рынка госзакупок и академического сектора |
| WCAG A на MVP с отложенным AA до P1 | Enterprise и госзаказчики блокируют участие в тендерах при A на MVP. AA для 6 критических сценариев добавляет ~30% трудозатрат frontend, но открывает госсектор, ФЗ № 419-ФЗ, Section 508 и European Accessibility Act с первого релиза |

## Последствия

**Положительные последствия:**
- Соответствие **ФЗ № 419-ФЗ** (Россия) с первого релиза
- Соответствие **Section 508** (США) для федеральных закупок
- Соответствие **European Accessibility Act (EAA)** для рынка ЕС
- Охват 70% пользователей с ограничениями
- Участие в тендерах без фазы доработки accessibility

**Отрицательные последствия:**
- Увеличение трудозатрат на ~30% для frontend (основная — альтернативная навигация графа, ARIA для кастомных компонентов, keyboard-only тесты)
- Необходимость ручного accessibility-тестирования перед каждым релизом
- Часть визуальных эффектов (3D-граф, D&D) требует альтернативных реализаций

**Меры снижения рисков:**
- Автоматизация проверок через axe-core в CI (блокировка critical/serious violations)
- Табличный fallback для 3D-графа как альтернативная навигация
- Button-based интерфейс для drag-and-drop операций
- Фиксация PASS/FAIL критерия: ≤ 3× времени обычного пользователя без мыши
- Детальная матрица сценариев в `human/artifacts/wcag-mvp-scenarios.md`

---
