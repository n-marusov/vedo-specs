# Автоматизация проверки миграции

| Атрибут | Значение |
|---------|----------|
| **ID** | REQ-NFR.DATA.migration-automation |
| **Уровень** | NFR |
| **Атрибут качества** | Supportability |
| **Приоритет** | P1 |
| **Статус** | УТВЕРЖДЕНО |
| **Источник** | — |
| **Критерии приёмки** | — |

---

## CI/CD-конвейер

Для сборки и проверки миграции нужен pipeline, который:

1. Запускается на release-ветке мажорной версии.
2. Поднимает test environment.
3. Выполняет `vedo-cli migrate apply --version <target>` или совместимый wrapper поверх `migrate-major.sh`.
4. Выполняет `vedo-cli migrate verify` или совместимый wrapper поверх `verify-integrity.sh`.
5. Фиксирует измеренный downtime.

`vedo-cli` является целевым административным интерфейсом миграций. Скрипты `migrate-major.sh` и `verify-integrity.sh` могут оставаться внутренней реализацией, но пользовательский и CI/CD контракт должен выражаться через `vedo-cli migrate plan/apply/verify/rollback`.

## Пример задания GitHub Actions

```yaml
name: Test Major Migration

on:
  push:
    branches: ["v2.0-release"]

jobs:
  test-migration:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - name: Start test environment
        run: docker-compose -f docker-compose.migration-test.yaml up -d
      - name: Run migration script
        run: ./scripts/migrate-major.sh --dry-run=false
      - name: Verify integrity
        run: ./scripts/verify-integrity.sh
      - name: Measure downtime
        run: echo "Downtime: $(cat /tmp/downtime.txt) sec"
```

## Открытые вопросы

- Нет по M2.1.
