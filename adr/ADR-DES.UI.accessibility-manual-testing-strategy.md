# ADR-DES.UI.accessibility-manual-testing-strategy

**Дата:** 2026-05-13  
**Статус:** Принято

## Контекст

WCAG 2.1 AA нельзя доказать только automated scan. FTC/accessiBe lessons показывают, что overlay-виджеты и автоматические проверки не подтверждают доступность реальных пользовательских сценариев.

## Требование-источник
- [accessibility.md](requirements/REQ-NFR.UI.accessibility.md)
- [user-stories.md](../user-stories.md)

## Решение

Для каждого релиза ввести закрытый перечень accessibility critical user scenarios A1–A12 с соответствующими пользовательскими историями US-a11y.* (US-a11y.navigation.tree-reader, US-a11y.navigation.graph-fallback, US-a11y.search.fulltext-keyboard, US-a11y.classes.create-reader, US-a11y.properties.create-object, US-a11y.individuals.edit-dynamic, US-a11y.comments.view-add, US-a11y.versioning.commit-history, US-a11y.i18n.switch-language, US-browse.public.view-accessible, US-a11y.account.close-confirm, US-a11y.auth.authentication), покрывающими навигацию, редактирование, комментарии, историю, переключение языка, публичный просмотр, закрытие аккаунта и OAuth.

Ручная проверка каждого сценария через screen reader (NVDA/VoiceOver) и keyboard-only доказывает WCAG 2.1 AA в реальных пользовательских сценариях, а time-to-task ≤ 3× норматива обычного пользователя даёт измеримый критерий успеха — в отличие от automated scan, который не проверяет контекст и динамические компоненты онтологического редактора.

A1–A3: навигация по дереву и графу через табличный запасной вариант; A4–A6: создание класса, ObjectProperty, редактирование индивида; A7–A12: комментарии, diff, переключение языка, публичный просмотр, закрытие аккаунта, OAuth.

## Рассмотренные альтернативы

| Альтернатива | Причина отклонения |
|--------------|--------------------|
| Только axe/Lighthouse | Не проверяет реальную навигацию, контекст и динамические компоненты |
| Наложенный accessibility widget | Не заменяет нативную доступность компонентов |
| Общий перечень сценариев без ID | Невоспроизводимо и не даёт измеримого coverage |
| Проверять только 3 сценария | Недостаточно для онтологического редактора с графом, diff, OAuth и destructive flows |

## Последствия

**Положительные последствия:**
- Доступность получает воспроизводимые release gates.
- P0-сценарии нельзя пропустить без блокировки релиза.
- Пользовательские истории напрямую трассируют accessibility coverage.

**Отрицательные последствия:**
- Ручной аудит увеличивает стоимость релиза.
- Нужно поддерживать screen reader compatibility для сложных UI-компонентов.

**Меры снижения рисков:**
- Automated scan остаётся вспомогательным быстрым gate.
- 3D-граф имеет табличный запасной вариант вместо попытки сделать canvas основным доступным интерфейсом.

---
