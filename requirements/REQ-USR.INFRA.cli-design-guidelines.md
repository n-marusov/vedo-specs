# CLI Design Guidelines for VEDO Core

| Атрибут | Значение |
|---------|----------|
| **ID** | REQ-USR.INFRA.cli-design-guidelines |
| **Уровень** | USR |
| **Атрибут качества** | Usability |
| **Приоритет** | P1 |
| **Статус** | УТВЕРЖДЕНО |
| **Источник** | — |
| **Критерии приёмки** | — |

---


## Общие принципы

1. **Все длительные операции (> 5 секунд) имеют индикацию прогресса.**
2. **Цветовой вывод используется только если stdout — TTY.**
3. **Exit codes соответствуют стандартам (0 = успех, 1 = общая ошибка, 130 = SIGINT).**
4. **Поддержка `--quiet` и `--verbose` — для всех команд.**

## Прогресс-бары

| Библиотека | Язык | Рекомендация |
|------------|------|--------------|
| `progressbar` | Go | `github.com/schollz/progressbar` |
| `indicatif` | Rust | `indicatif::ProgressBar` |
| `tqdm` | Python | `from tqdm import tqdm` |

**Пример использования (Go):**
```go
bar := progressbar.DefaultBytes(totalBytes, "Экспорт триплетов")
for _, chunk := range chunks {
    bar.Add(len(chunk))
    // обработка
}
```

## Спиннеры

| Библиотека | Язык | Рекомендация |
|------------|------|--------------|
| `spinner` | Go | `github.com/briandowns/spinner` |
| `indicatif` | Rust | `indicatif::ProgressBar` с `spinner` стилем |
| `halo` | Python | `halo.Halo` |

## Цвета

| Статус | Цвет | ANSI код |
|--------|------|----------|
| Успех (✓) | Зелёный | `\033[32m` |
| Ошибка (✗) | Красный | `\033[31m` |
| Предупреждение (⚠) | Жёлтый | `\033[33m` |
| Информация (ℹ) | Синий | `\033[34m` |
| Сброс | — | `\033[0m` |

**Проверка TTY:**
```go
if fileInfo, _ := os.Stdout.Stat(); (fileInfo.Mode() & os.ModeCharDevice) != 0 {
    // TTY, можно использовать цвета
}
```

## Verbosity уровни

| Уровень | Флаг | Вывод |
|---------|------|-------|
| 0 | `--quiet` (-q) | Только ошибки (stderr) |
| 1 | (по умолчанию) | Прогресс, основные этапы, итоговый результат |
| 2 | `--verbose` (-v) | Детальный вывод, технические параметры |
| 3 | `--verbose --verbose` (-vv) | Очень детальный (debug) |

## Dry-run

Для destructive операций обязателен флаг `--dry-run`:

```bash
$ vedo-cli decommission tenant --id tenant-456 --dry-run
```

**Вывод:** Что будет удалено, без фактических изменений.

## Graceful shutdown

Обработка SIGINT (Ctrl+C):

```go
c := make(chan os.Signal, 1)
signal.Notify(c, os.Interrupt, syscall.SIGTERM)
go func() {
    <-c
    fmt.Println("\n⚠ Операция прервана. Сохраняем состояние...")
    // сохранение состояния
    os.Exit(130)
}()
```

## Примеры

### Хороший CLI вывод

```bash
$ vedo-cli ontology export --id ont-123

Экспорт онтологии "Корпоративная модель"
┌─────────────────────────────────────────────────────────────┐
│ ████████████████████████████████████████░░░░░░░░░░ 78%     │
│ Обработано: 78 432 / 100 000 триплетов                     │
│ Скорость: 3 210 триплетов/сек                              │
│ Осталось: ~7 секунд                                         │
└─────────────────────────────────────────────────────────────┘
✅ Экспорт завершён: ont-123-20260523-104512.ttl (12 MB)
```

### Плохой CLI вывод (чего следует избегать)

```bash
$ vedo-cli ontology export --id ont-123
Экспорт...
OK.
(нет информации о прогрессе, объёме, времени)
```

## Связанные артефакты

| Артефакт | Связь |
|----------|-------|
| `vedo-cli-specification.md` | Общая спецификация CLI |
| `decommission-usability.md` | Требования к прогрессу decommission |
| `migration-runbook.md` | Требования к прогрессу миграции |
| `ADR-IMPL.STACK.vedo-cli-framework-strategy` | ADR с UX Requirements for CLI |
