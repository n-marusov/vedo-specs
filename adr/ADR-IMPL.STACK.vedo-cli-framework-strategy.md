# ADR-IMPL.STACK.vedo-cli-framework-strategy

**Дата:** 2026-05-19  
**Статус:** PROPOSED → ACCEPTED

## Контекст

В `ADR-IMPL.STACK.vedo-cli-language-strategy` принят Go как язык реализации `vedo-cli`. Однако выбор конкретного фреймворка для построения командной структуры не был зафиксирован. Milestone `002-security-and-cli-foundation`, Stage 2 включает реализацию `vedo-cli` command framework, что требует явного архитектурного решения.

`vedo-cli` — единая административная CLI-утилита экосистемы VEDO Core. Ключевые функциональные требования к фреймворку (согласно vision.md и vedo-cli-specification.md):

- **Глубокая вложенность команд:** `vedo-cli backup create --full`, `vedo-cli backup restore --id <id>`, `vedo-cli diagnose trace --id <trace_id>`. Фреймворк должен естественно поддерживать root → sub → subsub иерархию.
- **Persistent flags:** `--verbose`, `--config`, `--output-format` должны быть доступны во всех подкомандах без дублирования.
- **Автодополнение:** bash/zsh completion для ускорения работы администраторов.
- **Автогенерация документации:** Markdown-документация команд для интеграции с Antora.
- **Структурированный `--help`:** Секции Usage, Flags, Examples, Subcommands.
- **Machine-readable output:** `--format json` / `--format yaml` для CI/CD интеграции.
- **Стандартизация:** код должен быть читаемым, поддерживаемым и следовать общепринятым практикам Go-сообщества.

## Требование-источник

- `ADR-IMPL.STACK.vedo-cli-language-strategy` — Go как язык реализации
- `human/milestones/002-security-and-cli-foundation/stage_2.md` — Stage 2: vedo-cli framework
- `human/artifacts/requirements/REQ-FUN.INFRA.vedo-cli-specification.md`
- `human/contracts/vedo-cli.md` — CLI-OPS-001

## Решение

Принять **cobra** (github.com/spf13/cobra) как фреймворк для построения команд `vedo-cli`.

`cobra` обеспечивает нативную поддержку глубокой вложенности команд через структуру `Command` с `AddCommand()`, встроенные persistent flags (флаг, объявленный в родительской команде, доступен во всех дочерних), автодополнение для bash/zsh/fish/powershell одной строкой (`completionCmd.Run`), генерацию man-страниц и Markdown-документации через `cobra/doc`, и структурированный help с секциями Usage, Aliases, Examples, Flags, Subcommands.

`cobra` является де-факто стандартом для сложных CLI в Go-экосистеме — используется в Kubernetes (`kubectl`), Docker (`docker`), GitHub CLI (`gh`), Hugo, Helm и множестве других инструментов. Это означает большой объём документации, примеров и best practices, а также熟悉ость для новых разработчиков, уже сталкивавшихся с cobra.

Использовать `cobra-cli init` для генерации каркаса, следовать стандартной структуре `cmd/` с файлом `root.go` и отдельными файлами на каждую подкоманду.

**Пример структуры команд vedo-cli (иллюстрация вложенности):**

```
vedo-cli
├── backup
│   ├── create (--full, --incremental, --verify)
│   ├── list
│   ├── restore (--id, --point-in-time, --verify)
│   └── verify (--id)
├── migrate
│   ├── apply (--version, --dry-run, --pre-backup)
│   └── rollback (--step, --force)
├── ontology
│   ├── import (--file, --format turtle|rdfxml)
│   ├── export (--id, --format, --output)
│   └── diff (--left, --right, --format)
├── diagnose
│   └── trace (--id, --time-range)
├── tenant
│   ├── create (--name, --sla standard|high|premium)
│   ├── delete (--id, --force, --preserve-backup)
│   └── seed (--profile canonical)
└── airgap
    └── prepare (--output, --include-docs, --verify)
```

## Рассмотренные альтернативы

**Альтернатива A: urfave/cli**

| Критерий | cobra | urfave/cli |
|----------|-------|------------|
| Вложенность | ✅ Естественная (AddCommand) | ⚠️ До 2 уровней, при 3+ код усложняется |
| Persistent flags | ✅ Встроенные | ⚠️ Через AppMetadata + ручная передача |
| Автодополнение | ✅ Встроенное (bash/zsh/fish/pwsh) | ⚠️ Ручная настройка через `cli.AutocompleteApp` |
| Автодокументация | ✅ `cobra/doc` (man, markdown) | ❌ Нет встроенной |
| Популярность | Очень высокая (k8s, docker, gh, hugo) | Средняя |
| Структура кода | Файл на команду, `RunE` callback | Flags внутри Action-функции |

urfave/cli проще для плоских CLI с 1-2 уровнями вложенности, но для `vedo-cli` с 3-4 уровнями (`vedo-cli backup restore --id`) код становится переусложнённым. Отсутствие встроенной автодокументации — критично для Antora-интеграции.

**Альтернатива B: Go flags + ручная реализация**

- + Нет внешних зависимостей, минимальный размер бинарника.
- − Ручная реализация парсинга подкоманд: `os.Args[1]` → switch/case на каждом уровне.
- − Ручная реализация persistent flags: копирование флагов в каждую подкоманду.
- − Ручная реализация автодополнения: генерация completion-скриптов вручную.
- − Нет автодокументации.
- − Высокий риск ошибок при расширении командной структуры.

Вариант приемлем для одноуровневого CLI (2-3 команды), но не для `vedo-cli` с десятками команд и глубиной до 4 уровней.

**Альтернатива C: Bubbletea (TUI)**

- + Интерактивный режим (меню, прогресс-бары для backup/migrate).
- − Архитектурно избыточна: административная CLI не требует интерактивного TUI.
- − Усложняет автоматизацию в CI/CD (нельзя просто `vedo-cli backup create`, нужен эмулятор терминала).
- − Не подходит для air-gapped сценариев и SSH-only операций.
- − Значительно больше зависимостей и размер бинарника.

Bubbletea может быть полезна для вспомогательных интерактивных утилит, но для `vedo-cli` как CI/CD-ready, non-interactive CLI она избыточна.

## Последствия

**Положительные последствия:**

- Естественная иерархия: каждая команда — отдельный файл в `cmd/`, иерархия строится через `AddCommand()`.
- Persistent flags (`--verbose`, `--config`, `--output-format`) наследуются всеми подкомандами без дублирования кода.
- Автодополнение для bash/zsh/fish/powershell — одна команда `vedo-cli completion bash > /etc/bash_completion.d/vedo-cli`.
- Автодокументация: генерация Markdown-документации через `cobra/doc` для Antora без ручного написания.
- Структурированный `--help` с Usage, Flags, Examples, Subcommands — из коробки.
- Большое сообщество: документация, примеры, best practices (k8s, docker, gh, hugo — все на cobra).
- `cobra-cli init` генерирует каркас проекта, ускоряя старт.

**Отрицательные последствия:**

- Больше boilerplate кода по сравнению с urfave/cli (каждая команда требует отдельный `Command` struct с настройкой).
- Немного медленнее компиляция из-за дополнительных зависимостей (но некритично для CLI, собираемого раз в релиз).
- Зависимость от внешнего пакета (spf13/cobra) — хотя это одна из наиболее стабильных и обратно-совместимых библиотек в Go-экосистеме.

**Меры снижения рисков:**

- Использовать `cobra-cli init` для генерации каркаса `vedo-cli`.
- Следовать стандартной структуре `cmd/`:
  ```
  llm/src/vedo-cli/
  ├── main.go
  ├── cmd/
  │   ├── root.go          # root command + persistent flags
  │   ├── backup.go        # backup parent command
  │   ├── backup_create.go # backup create subcommand
  │   ├── backup_restore.go
  │   ├── migrate.go
  │   ├── diagnose.go
  │   └── ...
  └── internal/            # business logic
      ├── backup/
      ├── diagnose/
      └── ...
  ```
- Вынести общие persistent flags (`--verbose`, `--config`, `--output-format`) как переменные в `root.go`.
- Использовать `RunE` (возвращает ошибку) вместо `Run` для единообразной обработки ошибок.
- Покрыть интеграционными тестами golden file tests для `--help` и completion.

## UX Requirements for CLI (дополнение)

### Progress indication

Все длительные операции (> 5 секунд) должны отображать прогресс.

| Тип операции | Тип индикации | Пример |
|--------------|---------------|--------|
| С известным объёмом (экспорт, импорт) | Прогресс-бар (проценты) | `[████████░░] 80% (80 000 / 100 000)` |
| С неизвестным объёмом (decommission tenant) | Спиннер + текущее действие | `⠋ Удаление узлов Neo4j...` |
| Многоэтапная (backup, миграция) | Пошаговая индикация | `[2/4] Upload to S3... ✅` |

### Rich feedback

| Элемент | Условие | Пример |
|---------|---------|--------|
| Цвета | stdout — TTY | `✓` (зелёный), `✗` (красный), `⚠` (жёлтый) |
| Спиннеры | Операция без известного прогресса | `⠋⠙⠹⠸⠼⠴⠦⠧⠇⠏` |
| Оценка времени | На основе скорости обработки | `Осталось ~15 секунд` |

### Verbosity levels

| Флаг | Назначение | Выход |
|------|------------|-------|
| `--quiet` (или `-q`) | Только ошибки (машиночитаемый) | Только сообщения об ошибках, exit code |
| `--verbose` (или `-v`) | Детальный вывод | Логи каждого действия, параметры, время выполнения |
| По умолчанию (без флагов) | Информативный, но лаконичный | Прогресс, этапы, итоговый результат |

### Graceful shutdown

При нажатии Ctrl+C (SIGINT):
1. Завершить текущую операцию до безопасной точки.
2. Сохранить состояние (если возможно).
3. Вывести сообщение: `⚠ Операция прервана. Временные файлы сохранены в /tmp/vedo-...`
4. Завершиться с exit code 130 (стандарт для SIGINT).

### Требования к реализации в cobra

```go
// Пример прогресс-бара с использованием github.com/schollz/progressbar
progress := progressbar.DefaultBytes(totalBytes, "Экспорт триплетов")

// Пример спиннера с использованием github.com/briandowns/spinner
spinner := spinner.New(spinner.CharSets[9], 100*time.Millisecond)
spinner.Suffix = " Удаление узлов Neo4j..."
spinner.Start()
```

**Примечание:** Выбор конкретных библиотек остаётся за разработчиком. Единые стандарты оформления CLI зафиксированы в `cli-design-guidelines.md`.

## Связанные ADR

- `ADR-IMPL.STACK.vedo-cli-language-strategy` — Go как язык реализации vedo-cli
- `ADR-DES.INFRA.vedo-cli-admin-boundary` — роль vedo-cli в экосистеме VEDO Core
- `ADR-DES.INFRA.vedo-cli-diagnostics-entrypoint` — vedo-cli как точка входа диагностики
- `ADR-IMPL.SECURITY.vedo-cli-credentials-strategy` — стратегия получения credentials для vedo-cli
- `human/artifacts/requirements/REQ-USR.INFRA.cli-design-guidelines.md` — единые стандарты оформления CLI

---
