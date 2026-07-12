# CSS-переменные кастомизации темы — VEDO Core

> **Производный артефакт.** Единый источник дизайна: `design/frontend.pen` (Pencil). Изменения вносятся только в Pencil; этот файл синхронизируется автоматически.
> Переменные доступны для переопределения в on-premise и приватном облаке через `customer-overrides/theme.css`.

Все переменные имеют light и dark тему. Значения по умолчанию указаны для light-темы.

---

## 1. Цветовая схема

| CSS-переменная | Токен | Light | Dark | Назначение |
|----------------|-------|-------|------|------------|
| `--background` | `color.background` | `#f7faf8` | `#0a0a0a` | Основной фон страниц |
| `--foreground` | `color.foreground` | `#0b1220` | `#fafafa` | Основной текст |
| `--card` | `color.card` | `#ffffff` | `#101010` | Фон карточек, сайдбаров |
| `--card-foreground` | `color.card-foreground` | `#0b1220` | `#fafafa` | Текст в карточках |
| `--popover` | `color.popover` | `#ffffff` | `#111111` | Фон поповеров, меню |
| `--popover-foreground` | `color.popover-foreground` | `#0b1220` | `#fafafa` | Текст в поповерах |
| `--muted` | `color.muted` | `#e7eeea` | `#141414` | Фон при наведении |
| `--muted-foreground` | `color.muted-foreground` | `#334155` | `#6b7280` | Второстепенный текст |
| `--accent` | `color.accent` | `#e3efe9` | `#171717` | Акцентный фон |
| `--accent-foreground` | `color.accent-foreground` | `#0b1220` | `#fafafa` | Текст на accent |
| `--primary` | `color.primary` | `#0f766e` | `#10b981` | Брендовый цвет (кнопки, ссылки) |
| `--primary-foreground` | `color.primary-foreground` | `#ecfeff` | `#0a0a0a` | Текст на primary |
| `--secondary` | `color.secondary` | `#e8eeea` | `#141414` | Вторичный фон (табы, чипы) |
| `--secondary-foreground` | `color.secondary-foreground` | `#0b1220` | `#fafafa` | Текст на secondary |
| `--destructive` | `color.destructive` | `#dc2626` | `#ef4444` | Опасные действия (удаление) |
| `--destructive-foreground` | `color.destructive-foreground` | `#fafafa` | `#fafafa` | Текст на destructive |
| `--border` | `color.border` | `#b7c4bc` | `#2a2a2a` | Границы и разделители |
| `--input` | `color.input` | `#b7c4bc` | `#2a2a2a` | Границы полей ввода |
| `--ring` | `color.ring` | `#0f766e` | `#10b981` | Outline фокуса |
| `--agent` | `color.agent` | `#7c3aed` | `#8b5cf6` | Цвет AI-агента |
| `--agent-foreground` | `color.agent-foreground` | `#5b21b6` | `#c4b5fd` | Текст AI-агента |
| `--info` | `color.info` | `#2563eb` | `#3b82f6` | Информационный (синий) |
| `--info-foreground` | `color.info-foreground` | `#1e40af` | `#dbeafe` | Текст на info |
| `--success` | `color.success` | `#059669` | `#10b981` | Успех (зелёный) |
| `--success-foreground` | `color.success-foreground` | `#065f46` | `#d1fae5` | Текст на success |
| `--warning` | `color.warning` | `#d97706` | `#f59e0b` | Предупреждение (оранжевый) |
| `--warning-foreground` | `color.warning-foreground` | `#92400e` | `#fef3c7` | Текст на warning |
| `--tool` | `color.tool` | `#0891b2` | `#06b6d4` | Цвет инструментов (циан) |
| `--tool-foreground` | `color.tool-foreground` | `#155e75` | `#a5f3fc` | Текст инструментов |
| `--tool-error` | `color.tool-error` | `#dc2626` | `#ef4444` | Ошибка инструмента |

## 1.1. Приоритеты

| CSS-переменная | Токен | Light | Dark | Назначение |
|----------------|-------|-------|------|------------|
| `--priority-urgent` | `color.priority-urgent` | `#dc2626` | `#ef4444` | Срочный приоритет |
| `--priority-high` | `color.priority-high` | `#ea580c` | `#f97316` | Высокий приоритет |
| `--priority-medium` | `color.priority-medium` | `#d97706` | `#f59e0b` | Средний приоритет |
| `--priority-low` | `color.priority-low` | `#0891b2` | `#06b6d4` | Низкий приоритет |

## 2. Статусы workflow

| CSS-переменная | Токен | Light | Dark | Назначение |
|----------------|-------|-------|------|------------|
| `--status-backlog` | `color.status.backlog` | `#6b7280` | `#6b7280` | Статус: бэклог |
| `--status-planning` | `color.status.planning` | `#7c3aed` | `#8b5cf6` | Статус: планирование |
| `--status-implementing` | `color.status.implementing` | `#d97706` | `#f59e0b` | Статус: реализация |
| `--status-review` | `color.status.review` | `#2563eb` | `#3b82f6` | Статус: ревью |
| `--status-ready` | `color.status.ready` | `#0891b2` | `#06b6d4` | Статус: готово |
| `--status-done` | `color.status.done` | `#059669` | `#10b981` | Статус: завершено |
| `--status-failed` | `color.status.failed` | `#dc2626` | `#ef4444` | Статус: провал |

## 3. Типографика

| CSS-переменная | Значение | Назначение |
|----------------|----------|------------|
| `--font-family-sans` | `Inter, system-ui, -apple-system, sans-serif` | Основной шрифт |
| `--font-family-mono` | `'IBM Plex Mono', 'JetBrains Mono', 'Fira Code', monospace` | Моноширинный (код) |
| `--font-size-4xs` | `9px` | Третичный текст |
| `--font-size-3xs` | `10px` | Мелкий текст |
| `--font-size-2xs` | `11px` | Подписи |
| `--font-size-sm` | `14px` | Мелкий текст |
| `--font-size-base` | `16px` | Основной текст |
| `--font-size-lg` | `18px` | Заголовки 3 уровня |
| `--font-size-xl` | `20px` | Заголовки 2 уровня |
| `--font-size-2xl` | `24px` | Заголовки 1 уровня |
| `--font-size-3xl` | `30px` | Крупные заголовки |

## 4. Механизм переопределения

On-premise: файл `customer-overrides/theme.css` загружается после основных стилей. Все переменные обязаны иметь fallback (значение по умолчанию).

```css
:root {
  --primary: #1E40AF;
  --background: #F8FAFC;
  --font-family-sans: 'Raleway', system-ui, sans-serif;
}
```

---

## 5. История изменений

| Версия | Дата | Изменение |
|--------|------|-----------|
| 2.0 | 2026-05-23 | Переписана под актуальные токены из Pencil (shadcn-совместимые имена, emerald primary, theme-aware) |
| 1.0 | 2026-05-18 | Начальная версия (Tailwind-based, blue primary) |
