# ADR-IMPL.SECURITY.bola-bfla-negative-tests-mandate

**Дата:** 2026-05-16  
**Статус:** Принято

## Контекст

**Проблема:** OWASP API Security Top 10 (2023) ставит BOLA (Broken Object Level Authorization) на первое место. VEDO Core — multi-tenant SaaS-платформа управления онтологиями, где каждый запрос содержит объектные ID (`tenantId`, `ontologyId`, `branchId`, `commitId`, `classId`, `individualId`, `propertyId`). Подмена любого из этих ID позволяет атакующему получить доступ к чужому tenant, чужой онтологии или совершить операцию с превышением привилегий.

**Индустриальные инциденты:**

| Компания | Год | Уязвимость | Ущерб |
|----------|-----|------------|-------|
| **GitLab** | 2023 | IDOR в issue attachments — пользователь мог читать файлы из любого проекта | Экспозиция данных всех проектов |
| **Uber** | 2022 | BOLA в API управления поездками — подмена rideId давала доступ к чужим заказам | Утечка данных водителей и пассажиров |
| **Facebook** | 2020 | BOLA в `pages/{page_id}/settings` — пользователь мог читать настройки чужих страниц | Утечка конфигурации бизнес-страниц |

**Существующие меры и их недостаточность:**

1. **Авторизация в коде** — каждый endpoint должен проверять права, но отсутствует метрика, какие эндпоинты покрыты negative-тестами, а какие нет. Code review пропускает BOLA в ~30% случаев (OWASP Benchmark).
2. **Code review** — человеческая ошибка неизбежна. OWASP Benchmark показывает, что ручной review находит ~60% BOLA-уязвимостей.
3. **Корректная архитектура (JWT, scopes)** — не защищает от подмены объектного ID внутри одного scope.
4. **SAST-анализ** — генерирует 30-50% false positives и не ловит runtime-логику (например, проверку tenantId в middleware).

**Почему это архитектурное решение, а не просто best practice:**

- Требует изменения CI/CD pipeline (новый security stage с negative тестами, SAST, coverage check)
- Влияет на все 12+ endpoint-классов REST, GraphQL, WebSocket, SPARQL
- Меняет подход к разработке: TDD для security (negative тесты пишутся одновременно с кодом)
- Требует инфраструктуры: изолированные test tenant, тестовые пользователи, resetable fixtures
- Вводит runtime monitoring с автоматической блокировкой при аномалиях
- Имеет архитектурные альтернативы, которые были рассмотрены и отклонены

## Требование-источник

- [BOLA/BFLA Negative Authorization Tests Specification v1.0](bola-bfla-negative-tests.md) — детальная техническая спецификация
- `human/constraints/security.yaml` — глобальная политика безопасности (applies_to: all)
- [ADR-DES.SECURITY.gitlab-like-organization-model](#adr-dessecuritygitlab-like-organization-model) — ролевая модель для BFLA тестов
- [ADR-DES.API.protocol-stack-strategy](#adr-desapiprotocol-stack-strategy) — полный перечень endpoint-классов
- [ADR-IMPL.PROCESS.gitlab-ci-cd-strategy](#adr-implprocessgitlab-ci-cd-strategy) — CI pipeline, в который интегрируется security stage

## Решение

**Принцип:** "Zero trust for object-level authorization testing" — каждый endpoint с объектными ID должен иметь negative-тест, доказывающий, что доступ без прав возвращает 403.

**Обязательные endpoint-классы для negative тестирования:**

| № | Endpoint класс | Объектные ID | Приоритет |
|---|---------------|--------------|-----------|
| 1 | `/api/v1/tenants/{tenantId}/*` | `tenantId` | P0 |
| 2 | `/api/v1/ontologies/{ontologyId}/*` | `ontologyId`, `tenantId` | P0 |
| 3 | GraphQL мутации (с объектными ID) | `ontologyId`, `classId`, `individualId`, `propertyId` | P0 |
| 4 | GraphQL запросы (с `ontologyId`) | `ontologyId` | P0 |
| 5 | `/api/v1/branches/{branchId}/*` | `branchId`, `ontologyId` | P0 |
| 6 | `/api/v1/commits/{commitId}/*` | `commitId`, `ontologyId`, `branchId` | P0 |
| 7 | SPARQL (с `ontologyId` через `default-graph-uri`) | `ontologyId` | P1 |
| 8 | `/api/v1/users/{userId}/*` (admin) | `userId` | P0 |
| 9 | `/api/v1/roles/*` (admin) | `roleId`, `tenantId` | P0 |
| 10 | `/api/v1/invites/{inviteId}` | `inviteId` | P1 |
| 11 | WebSocket CollaborationService (join room) | `ontologyId` | P0 |
| 12 | `/api/v1/backups/*` (admin) | `backupId`, `tenantId` | P0 |

**Четыре обязательных типа negative тестов:**

| Тип | Название | Сценарий | Ожидаемый статус |
|-----|----------|---------|------------------|
| A | Cross-tenant BOLA | Пользователь Tenant A → объект Tenant B | 403 |
| B | Cross-object BOLA | Пользователь Tenant A → объект Tenant A без прав | 403 |
| C | BFLA (Privilege escalation) | Пользователь с низкой ролью → admin endpoint | 403 |
| D | IDOR (guessable IDs) | Подстановка предсказуемых ID (numeric+1, UUID variant) | 403 |

**Критерии PASS/FAIL:**

```
∀ t ∈ Tests(P0) : t.expected_status == 403 ∧ t.actual_status == 403
∀ t ∈ Tests(P0) : t.actual_status ∉ {404, 500}
∀ t ∈ Tests : t.audit_log_must_contain == true → ∃ audit_record
```

- 100% P0 тестов → 403, 0% → 404/500 — блокирует Merge Request и Release
- P1 тесты ≥ 90% проход → не блокирует MR, warning в CI

**Инструментарий (обязательный):**

| Инструмент | Назначение | Блокирует MR? |
|-----------|-----------|---------------|
| Integration tests (CI stage) | Прогон YAML-defined negative тестов каждого endpoint | Да (P0) |
| SAST (check-authz-coverage) | Поиск endpoints без авторизационной проверки | Да |
| API contract tests | Аннотации авторизации в OpenAPI/GraphQL schema | Да |
| Fuzzing (IDOR) | Перебор предсказуемых ID | Нет (warning) |
| DAST (staging) | Динамический анализ в pre-prod среде | Нет (warning) |

**CI pipeline integration:**

```
MR → SAST → Build → Deploy → Contract Tests → BOLA/BFLA Tests → Fuzzing → DAST
                                                                         ↓
                                              FAIL (P0) → блокировка MR
                                              WARN (P1) → security issue
```

**Runtime monitoring (production):**

| Сигнатура | Условие | Действие | Priority |
|-----------|---------|----------|----------|
| High 403 rate | Доля 403 > 5% на endpoint за 5 мин | P2 алерт | P2 |
| Sequential 403 sweep | >10 403 от одного IP на разные tenantId за 1 мин | Временная блокировка IP + P1 алерт | P1 |
| WAF tenant mismatch | tenantId в JWT ≠ tenantId в URL | Блокировка запроса + P0 алерт | P0 |

**Исключения** (не требуют negative тестов):
- Публичный Browse API (`/browse/*`) — read-only, без авторизации
- Health/ready endpoints — нет объектных ID
- OAuth/логин — pre-authentication
- `GET /openapi.json`, GraphQL introspection — публичные spec

## Рассмотренные альтернативы

**Альтернатива A (Baseline):** Полагаться только на code review и manual pentest

| Достоинства | Недостатки |
|-------------|------------|
| Нулевая стоимость внедрения | Человеческая ошибка: OWASP Benchmark — только ~60% detection rate |
| Не требует изменений в CI/CD | Не масштабируется: каждый pentest — недели работы |
| Не замедляет разработку | Не ловит регрессии: изменение в middleware может сломать авторизацию |

**Причина отклонения:** BOLA — #1 угроза API Security Top 10. Полагаться только на ручную проверку для multi-tenant SaaS неприемлемо.

**Альтернатива Б:** Только runtime detection (WAF + monitoring без negative тестов)

| Достоинства | Недостатки |
|-------------|------------|
| Детектирует атаки в production | Детектирует постфактум — данные уже скомпрометированы |
| Минимальное влияние на разработку | Не защищает от автоматизированных sweep-атак с низкой скоростью |
| Покрывает все endpoint-классы сразу | Не даёт метрики "какие endpoint-классы защищены" |

**Причина отклонения:** Runtime detection обнаруживает атаку, но не предотвращает. Negative тесты предотвращают уязвимости до production.

**Альтернатива В:** Только SAST статический анализ

| Достоинства | Недостатки |
|-------------|------------|
| Автоматизирован, запускается в CI | 30-50% false positives — команда привыкает игнорировать |
| Покрывает всю кодовую базу | Не ловит runtime-логику (проверка tenantId в middleware, conditional authorization) |

**Причина отклонения:** SAST необходим как complimentary инструмент, но недостаточен как единственная мера.

**Альтернатива Г:** Negative тесты только для критических endpoints (tenant + ontology)

| Достоинства | Недостатки |
|-------------|------------|
| Снижает затраты на тестирование на ~50% | Оставляет риск для веток, коммитов, GraphQL мутаций |
| Быстрее внедрить | BFLA на admin endpoints остаётся непокрытым |

**Причина отклонения:** Атака через менее очевидные endpoints — известная тактика обороны вглубь. Покрытие должно быть полным для всех P0 endpoint-классов.

**Альтернатива Д:** Автоматическая генерация negative тестов из OpenAPI + атрибутов авторизации

| Достоинства | Недостатки |
|-------------|------------|
| Минимизирует ручную работу | Требует мета-аннотаций во всех endpoint |
| Гарантирует 100% coverage по OpenAPI | OpenAPI не описывает бизнес-логику (какие tenantId разрешены) |

**Причина отклонения:** Слишком сложно для MVP. Рекомендовано как future enhancement.

## Последствия

**Положительные последствия:**

- Обнаружение BOLA/BFLA уязвимостей до production — не после атаки
- Регрессии в авторизации ловятся в CI (не катятся в prod)
- Соответствие OWASP ASVS V4 (Access Control) / V5 (Authorization) требованиям
- Уверенность при multi-tenant деплое — один tenant не может прочитать другой
- Документированный стандарт для разработчиков — как писать negative тесты
- Метрика coverage endpoint-классов negative тестами

**Отрицательные последствия:**

| Последствие | Описание | Мера снижения |
|-------------|----------|---------------|
| Увеличение времени разработки | ~20% overhead на написание negative тестов для нового endpoint | Тест-хелперы и фикстуры. YAML-шаблоны сокращают время до 5 мин на тест |
| CI pipeline медленнее | +3-5 минут на security stage | Параллельный запуск security stage. Non-blocking для P1 тестов |
| False positives | Тест падает, хотя авторизация правильная | Автоматический retry (3 попытки). Возможность пометить как flaky с review |
| Сложность тестирования WebSocket | JoinRoom — не HTTP, сложнее проверить статус | WebSocket-тест через подготовку фикстур + проверка audit log |
| SPARQL тесты (P1) | SPARQL endpoint требует ontology в запросе | P1 не блокирует MR, только warning |

**Compliance Mapping:**

| Стандарт | Требование | Реализация |
|----------|------------|------------|
| **OWASP ASVS 4.0.3 V4.1** | "Пользователь может получить доступ только к тем объектам, на которые у него есть права" | Negative тесты типов A, B, D для P0 endpoint-классов |
| **OWASP ASVS 4.0.3 V4.2** | "Защита от IDOR, ответ 403 а не 404" | Тип D (IDOR). Критерий: 0% тестов возвращают 404 |
| **OWASP ASVS 4.0.3 V5.1** | "Чёткое разделение ролей и проверка прав на каждый запрос" | Тип C (BFLA) для admin endpoints |
| **SOC2 CC6.1** | Logical access controls — предотвращение несанкционированного доступа | Negative тесты как evidence в CI. Runtime monitoring как detective control |
| **SOC2 CC7.1** | Monitoring — выявление аномальной активности | Runtime detection (sequential 403 sweep, high 403 rate) |

## Ссылки

- [BOLA/BFLA Negative Authorization Tests Specification v1.0](bola-bfla-negative-tests.md) — техническая реализация: YAML-шаблоны, примеры кода, CI-конфигурация, чек-лист code review
- [ADR-DES.SECURITY.gitlab-like-organization-model](#adr-dessecuritygitlab-like-organization-model) — ролевая модель (источник тестовых ролей для BFLA)
- [ADR-DES.API.protocol-stack-strategy](#adr-desapiprotocol-stack-strategy) — полный перечень endpoint-классов
- [ADR-IMPL.PROCESS.gitlab-ci-cd-strategy](#adr-implprocessgitlab-ci-cd-strategy) — CI/CD pipeline, target для security stage
- `human/constraints/security.yaml` — глобальная политика безопасности

---
