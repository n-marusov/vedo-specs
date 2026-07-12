# Gate-политика регрессий авторизации перед релизом

| Атрибут | Значение |
|---------|----------|
| **ID** | REQ-NFR.SECURITY.authorization-regression-gates |
| **Уровень** | NFR |
| **Атрибут качества** | Functionality |
| **Приоритет** | P0 |
| **Статус** | УТВЕРЖДЕНО |
| **Источник** | — |
| **Критерии приёмки** | — |

---

## Что описывает

Фиксирует блокирующие проверки авторизации (policy-as-code) перед выкладкой в production.

## Обязательные классы проверок

| Класс | Описание | Минимальное покрытие |
|---|---|---:|
| BOLA | Object-level access control | 100% критичных read/write endpoints |
| BFLA | Function-level access control | 100% административных и destructive операций |
| Tenant isolation | Межтенантные запреты доступа | 100% multi-tenant API путей |
| Policy drift | Проверка отклонений от эталонных RBAC/ABAC policy | 100% production policy bundles |

## Блокирующие условия release gate

Release в production блокируется, если:

- обнаружен >= 1 критический дефект авторизации;
- обнаружен >= 1 несанкционированный cross-tenant доступ;
- покрытие обязательных негативных тестов < 100%;
- policy bundle не подписан или signature verification не пройдена.

## Режим выполнения

- Проверки запускаются на каждом merge request в `main` и перед `deploy-prod`.
- Максимальная длительность полного authorization gate: <= 15 минут.
- При flaky-падении допускается 1 автоматический re-run; итоговый статус должен быть deterministically pass/fail.

## Артефакты проверки

- machine-readable отчет (`json`) с ID тестов и outcome;
- список затронутых endpoint/policy;
- подпись policy bundle;
- ссылка на pipeline/job и commit SHA.

Срок хранения отчетов: не менее 3 лет (WORM/append-only).

## Бизнес-правила

- Ручной override security gate запрещен вне break-glass процесса.
- Любой release без authorization gate считается невалидным и подлежит немедленному rollback.
- Изменения ролей/permission matrix обязаны сопровождаться обновлением негативных тестов в том же MR.
