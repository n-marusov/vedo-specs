# Автоматическое тестирование обратной совместимости API

| Атрибут | Значение |
|---------|----------|
| **ID** | REQ-NFR.API.compatibility-testing |
| **Уровень** | NFR |
| **Атрибут качества** | Functionality |
| **Приоритет** | P1 |
| **Статус** | УТВЕРЖДЕНО |
| **Источник** | — |
| **Критерии приёмки** | — |

---

## Цель

Тестирование совместимости API проверяет, что клиенты и тесты старой версии продолжают работать против новой major версии API.

## Pipeline CI

Pipeline должен:

1. Запускаться на pull request в `main` и release-ветки, например `v2-release`.
2. Поднимать окружения v1 и v2.
3. Запускать v1 API tests против v1.
4. Запускать v1 API tests против v2 для проверки compatibility.
5. Считать compatibility failures и публиковать предупреждение.

## Пример GitHub Actions workflow

```yaml
name: API Compatibility Test

on:
  pull_request:
    branches: ["main", "v2-release"]

jobs:
  test-compatibility:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - name: Start v1 and v2 environments
        run: docker-compose -f docker-compose.compat.yaml up -d
      - name: Run v1 API tests against v1
        run: ./api-tests/v1/run-tests.sh --url http://localhost:8081
      - name: Run v1 API tests against v2
        run: ./api-tests/v1/run-tests.sh --url http://localhost:8082
        continue-on-error: true
      - name: Check v1 to v2 compatibility ratio
        run: |
          COMPAT=`wc -l /tmp/v1_on_v2_failures.txt`
          if [ $COMPAT -gt 0 ]; then
            echo "$COMPAT tests failed on v2"
          fi
```

## Поведение CI

В примере compatibility failures не блокируют CI, но логируются. Это фиксирует риски, не останавливая разработку автоматически.

## Открытые вопросы

- Нет по M2.2.
