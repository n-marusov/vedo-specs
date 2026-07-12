# ADR-DES.UI.navigation-state-strategy

**Дата:** 2026-05-10  
**Статус:** Принято

## Контекст

VEDO Core работает с крупными онтологиями. Потеря текущего места в дереве/графе, сброс фильтров или сброс выбранного класса после WebSocket-обновления увеличивает когнитивную нагрузку и ухудшает usability.

## Требование-источник
- [graph-navigation.md](requirements/REQ-USR.UI.graph-navigation.md)

## Решение

Разделить domain data state и navigation/UI state через Navigation State Store с URL/session persistence.

Разделение гарантирует, что realtime-обновления (WebSocket) не сбрасывают выбранный узел, раскрытые ветки дерева, фильтры и скролл, а URL-синхронизация ключевых элементов позволяет deep link и восстановление сессии без потери контекста даже при работе с крупными онтологиями.

Domain entities хранить в Apollo cache; Navigation State Store управляет selected entity, expanded tree nodes, filters, active panel и scroll anchors; deep-link-важное состояние синхронизировать в URL, сессионное — в sessionStorage.

## Рассмотренные альтернативы

| Альтернатива | Причина отклонения |
|--------------|--------------------|
| Навигация без состояния | Не подходит для больших онтологий и долгих рабочих сессий |
| Хранить navigation state вместе с domain entities | Realtime updates могут случайно сбрасывать UI context |
| Всё состояние держать только в URL | URL становится перегруженным и хранит несущиеся детали UI |

## Последствия

**Положительные последствия:**
- Состояние навигации получает явного владельца.
- Realtime updates перестают быть причиной потери контекста.
- Deep links и восстановление сессии проектируются сознательно.

**Отрицательные последствия:**
- Нужно определить, какое состояние относится к URL, sessionStorage или памяти.
- Возможны stale references после удаления/переименования entity.

**Меры снижения рисков:**
- Ввести navigation state schema и migration policy.
- Обрабатывать missing selected entity через deleted/empty state.

---
