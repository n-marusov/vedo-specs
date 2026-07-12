# Политика идемпотентности write API

| Атрибут | Значение |
|---------|----------|
| **ID** | REQ-FUN.API.write-idempotency |
| **Уровень** | FUN |
| **Атрибут качества** | Interface |
| **Приоритет** | P0 |
| **Статус** | УТВЕРЖДЕНО |
| **Источник** | — |
| **Критерии приёмки** | — |

---

## Что описывает

Фиксирует обязательные требования к идемпотентности операций записи во внешнем API VEDO Core.

## Область действия

- Все внешние REST write endpoints (`POST`, `PUT`, `PATCH`, `DELETE`) под `/api/v1/...`.
- Все GraphQL mutations, изменяющие состояние.
- Исключения допускаются только для явно перечисленных async fire-and-forget операций.

## Обязательные требования

| Параметр | Требование |
|---|---|
| Header/API key | Обязательный `Idempotency-Key` для 100% критичных write операций |
| Окно дедупликации | 24 часа по умолчанию |
| Повтор с тем же ключом и payload | Должен вернуть тот же бизнес-результат и тот же `resource_id` |
| Повтор с тем же ключом и другим payload | `409 IDEMPOTENCY_KEY_REUSED_WITH_DIFFERENT_PAYLOAD` |
| Хранилище ключей | Durable store (PostgreSQL/Redis with persistence), не in-memory only |

## Покрытие и исключения

- Минимальное покрытие идемпотентностью для write endpoints уровня production: >= 99%.
- Для критичных write операций (изменение онтологии, merge, import/export control, membership/role changes): 100%.
- Допустимая доля временных исключений: <= 1% от общего числа write endpoints, с ADR-обоснованием и сроком устранения <= 30 дней.

## Проверка в CI/CD

- На каждом MR в `main` выполняется idempotency contract test suite.
- Blocker rule: если покрытие критичных write операций < 100%, MR блокируется.
- Blocker rule: если общее покрытие write операций < 99%, MR блокируется.
- Базовый отчет публикуется как `idempotency-coverage-report.json`.

## Наблюдаемость

- Метрика `idempotency_replay_total` по endpoint и tenant.
- Метрика `idempotency_conflict_total` по endpoint.
- Доля конфликтов ключей (`409`) > 0.5% за 24 часа требует investigation.

## Бизнес-правила

- Write endpoint без задокументированной idempotency-семантики запрещен.
- Изменение payload-схемы write endpoint обязано сопровождаться обновлением idempotency тестов в том же MR.
- Использование случайного/невалидного формата `Idempotency-Key` должно отклоняться кодом `400 INVALID_IDEMPOTENCY_KEY`.
