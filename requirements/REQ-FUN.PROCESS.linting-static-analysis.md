# Linting & Static Analysis Requirements v1.0

| Атрибут | Значение |
|---------|----------|
| **ID** | REQ-FUN.PROCESS.linting-static-analysis |
| **Уровень** | FUN |
| **Атрибут качества** | Implementation |
| **Приоритет** | P0 |
| **Статус** | УТВЕРЖДЕНО |
| **Источник** | — |
| **Критерии приёмки** | — |

---

## Что описывает

Документ фиксирует требования к статическому анализу и линтингу кодовой базы VEDO Core, определяет инструменты, правила, gate'ы и ответственность за соблюдение качества кода на каждом этапе разработки.

Цель: обеспечить единообразие кода, предотвратить дефекты на ранней стадии, автоматизировать проверку качества в CI/CD.

## 1) Инструменты по языкам

| Язык | Сервисы | Линтер | Конфигурация | Версия |
|------|---------|--------|-------------|--------|
| **TypeScript/Vue** | frontend, publish-browse-ui | Biome | `biome.json` | ≥ 1.9 |
| **Go** | api-gateway, auth-service, commenting-service, ticket-api, ticket-notifier, ticket-telemetry-listener, ticket-sync, support-service | golangci-lint | `.golangci.yml` | ≥ 1.59 |
| **Rust** | ontology-service, versioning-service, publisher-service, public-browse-api | Clippy (cargo) | `[lints.clippy]` в `Cargo.toml` | ≥ 1.77 |
| **Python** | metrics-service, ticket-classifier | Ruff | `[tool.ruff]` в `pyproject.toml` | ≥ 0.4 |

## 2) Правила линтинга

### 2.1 TypeScript/Vue (Biome + vue-tsc)

**Biome** (стиль и синтаксис):
- `recommended: true` — базовый набор правил Biome
- `noUnusedVariables: error` — неиспользуемые переменные запрещены (отключено для `*.vue`, см. ниже)
- `noUnusedImports: error` — неиспользуемые импорты запрещены
- `noExplicitAny: warn` — использование `any` требует обоснования
- `noConsoleLog: warn` — `console.log` только для отладки
- `useConst: error` — предпочитать `const` перед `let`
- `useImportType: error` — type-only импорты через `import type`
- `quoteStyle: single` — одинарные кавычки
- `semicolons: asNeeded` — точки с запятой только где необходимы
- `lineWidth: 100` — максимальная ширина строки

**vue-tsc** (проверка типов Vue SFC):
- `vue-tsc --noEmit` — авторитивная проверка типов для Vue Single File Components
- Проверяет типы в `<script setup>`, `<template>` выражениях, props, emits, slots
- Должен проходить успешно перед релизом (критичный gate)
- Biome не анализирует `<template>` scope, поэтому `noUnusedVariables` отключено для `**/*.vue` в `biome.json`; эту проверку выполняет `vue-tsc`

### 2.2 Go (golangci-lint)

Включённые линтеры:
- `errcheck` — проверка необработанных ошибок
- `gosimple`, `govet`, `staticcheck` — стандартные проверки
- `ineffassign`, `unused` — неиспользуемый код
- `gofmt`, `goimports` — форматирование и импорты
- `misspell` — орфографические ошибки
- `bodyclose`, `noctx` — ресурсные утечки
- `prealloc`, `exportloopref` — производительность
- `gocritic`, `revive` — дополнительные проверки
- `gocyclo` (max 15) — цикломатическая сложность
- `nakedret` (max 30) — запрет naked returns
- `copyloopvar` — копирование переменных цикла

Исключения:
- `_test.go` файлы: `errcheck`, `govet`, `unused` отключены
- `main.go`: `gochecknoglobals` отключён

### 2.3 Rust (Clippy)

- `pedantic: warn` — расширенный набор правил
- `missing_errors_doc: allow` — документация ошибок опциональна
- `missing_panics_doc: allow` — документация паник опциональна
- `module_name_repetitions: allow` — повторения имён модулей допустимы
- `must_use_candidate: allow` — `#[must_use]` не обязателен

### 2.4 Python (Ruff)

Конфигурация в `[tool.ruff]` секции `pyproject.toml`. Зависимости проекта и линтер объединены в единый файл.

Включённые правила:
- `E`, `W`, `F` — pycodestyle, Pyflakes
- `I` — isort (сортировка импортов)
- `N` — pep8-naming
- `UP` — pyupgrade (современный синтаксис)
- `B` — flake8-bugbear
- `SIM` — flake8-simplify
- `TCH` — type-checking импорты
- `RUF` — Ruff-specific правила
- `C4` — comprehensions
- `PT` — pytest style
- `RET` — return statements
- `ARG` — unused arguments
- `PTH` — pathlib usage
- `ERA` — commented-out code detection
- `PL` — Pylint rules
- `TRY` — exception handling best practices

Исключения:
- `E501` — длина строки (обрабатывается форматтером)
- `PLR0913` — слишком много аргументов
- `TRY003` — длинные сообщения исключений
- `tests/**`: `S101` (assert), `PLR2004` (magic values)

## 3) Gate'ы валидации

| Gate ID | Тип | Язык | Команда | Критерий прохода | Mandatory |
|---------|-----|------|---------|------------------|-----------|
| `GATE-LINT-GO-001` | lint | Go | `make lint-go` | 0 ошибок, 0 предупреждений | Да |
| `GATE-LINT-RUST-001` | lint | Rust | `make lint-rust` | 0 ошибок, 0 предупреждений | Да |
| `GATE-LINT-PYTHON-001` | lint | Python | `make lint-python` | 0 ошибок, 0 предупреждений | Да |
| `GATE-LINT-TYPESCRIPT-001` | lint | TypeScript | `make lint-typescript` | 0 ошибок, 0 предупреждений | Да |
| `GATE-TYPECHECK-VUE-001` | typecheck | Vue/TypeScript | `make typecheck-typescript` | 0 ошибок типов (`vue-tsc --noEmit`) | Да |

### 3.1 Когда выполняются gate'ы

Все lint-гейты выполняются:
- **После каждого этапа реализации**, изменяющего код приложения (`/implement`)
- **Перед запуском `/validate`** (release gate)
- **В CI/CD pipeline** на каждый PR/MR

### 3.2 Blocking conditions

MR/PR блокируется если:
- любой mandatory lint-gate вернул ошибки;
- код не отформатирован согласно конфигурации линтера;
- добавлены новые файлы без соответствующей конфигурации линтера.

## 4) Инфраструктура сборки

Makefile targets в `llm/src/build/`:

| Target | Файл | Описание |
|--------|------|----------|
| `lint-go` | `build/go.mk` | Запускает `golangci-lint run` для всех Go сервисов |
| `lint-rust` | `build/rust.mk` | Запускает `cargo clippy -- -D warnings` для всех Rust сервисов |
| `lint-python` | `build/python.mk` | Запускает `ruff check` для всех Python сервисов (через `pyproject.toml`) |
| `build-python` | `build/python.mk` | Устанавливает зависимости через `pip install -e .` (editable install) |
| `lint-typescript` | `build/typescript.mk` | Запускает `pnpm lint` (Biome CI) для всех TS сервисов |
| `typecheck-typescript` | `build/typescript.mk` | Запускает `vue-tsc --noEmit` для всех TS/Vue сервисов |

## 5) Форматирование vs линтинг

| Аспект | Форматирование | Линтинг |
|--------|---------------|---------|
| Цель | Единообразие стиля | Поиск дефектов и антипаттернов |
| Автофикс | Да (`biome check --write`, `ruff format`, `gofmt`) | Частично (`biome check --write`, `ruff check --fix`); `vue-tsc` — только проверка |
| Блокировка CI | Нет (автоприменимо) | Да (ошибки блокируют) |
| Конфигурация | Formatter секция в конфигах | Linter/rules секция в конфигах |

## 6) Ответственность

| Роль | Ответственность |
|------|----------------|
| Engineering Lead | Утверждение правил линтинга, обновление конфигов |
| Developer | Локальный запуск линтера перед коммитом, исправление ошибок |
| CI/CD | Автоматический запуск lint-gates на каждый PR/MR |
| Reviewer | Проверка что lint-gates прошли, ревью предупреждений |

## 7) Исключения и подавления

- Inline-подавления допустимы только с комментарием-обоснованием (`// biome-ignore`, `# noqa:`, `#[allow(...)]`, `//nolint`)
- File-level подавления требуют согласования с Engineering Lead
- Directory-level подавления (`per-file-ignores`) допустимы для тестов и генерированного кода

## 8) Статус

Решено.
