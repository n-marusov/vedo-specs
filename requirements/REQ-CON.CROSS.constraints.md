# Технические ограничения

| Атрибут | Значение |
|---------|----------|
| **ID** | REQ-CON.CROSS.constraints |
| **Уровень** | CON |
| **Атрибут качества** | — |
| **Приоритет** | P0 |
| **Статус** | УТВЕРЖДЕНО |
| **Источник** | — |
| **Критерии приёмки** | — |

---

## Производительность (из `human/constraints/performance.yaml`)

Единица масштаба для производительности/SLA: Canonical Workload Profile из `human/artifacts/requirements/REQ-NFR.PERF.canonical-workload-profile.md`.

| Метрика | Значение |
|---------|----------|
| Задержка p95 | 120 ms |
| Задержка p99 | 250 ms |
| Максимальная доля ошибок | 0.5% |
| SLO доступности | 99.9% для MVP SaaS; 99.95% для Enterprise production; 99.99% только Premium Enterprise по согласованию |
| Time-to-Diagnose P0 | ≤ 10 минут через `vedo-cli diagnose` |
| Time-to-Diagnose P1 | ≤ 30 минут через `vedo-cli diagnose` |

Валидация: 30 сек прогрева, 5 мин тестового окна, HDR histogram.

Политика SLO доступности зафиксирована в `human/artifacts/requirements/REQ-NFR.INFRA.availability-slo.md`.

## Безопасность (из `human/constraints/security.yaml`)

Критические правила:
- **prepared_statements_only**: Все запросы к БД — параметризованные
- **no_secrets_in_logs**: Секреты, токены, пароли не логируются
- **authn_required**: Все endpoints, изменяющие состояние, требуют аутентификации
- **oauth_providers**: Поддержка авторизации через российские OAuth провайдеры (ЕСИА/Госуслуги, Яндекс, VK) и Google
- **admin_operations_via_vedo_cli**: Прямой административный доступ к Neo4j/PostgreSQL/S3/MinIO в production допускается только через `vedo-cli` или согласованный break-glass процесс.
- **admin_audit_required**: Все действия `vedo-cli` логируются с actor, role, command, target environment, trace_id/correlation_id, result и redacted input summary.

Исключения: требуют одобрения команды безопасности и даты истечения (max 30 дней).

Подробнее: `human/artifacts/requirements/REQ-NFR.SECURITY.security-requirements.md` — хранение Shamir-частей Emergency Admin пароля и аудит break-glass доступа.

## Наблюдаемость (из `human/constraints/observability.yaml`)

Критические правила:
- **structured_logging_only**: Только JSON / tracing spans (без println, dbg!)
- **log_entry_exit**: Каждый endpoint логирует вход (params, request_id) и выход (status, duration)
- **log_all_errors**: Полный контекст ошибок (request_id, entity_id, input summary, details)
- **log_state_changes**: Все мутации (DB write, status transition, cache invalidation) логируются
- **log_external_calls**: Все исходящие вызовы (HTTP, gRPC, DB, queue) логируются
- **request_correlation**: Все log events содержат correlation ID (trace_id)
- **no_sensitive_in_logs**: PII, secrets, tokens маскируются
- **log_levels_correct**: error/warn/info/debug используются корректно

Исключения: требуют одобрения team lead и даты истечения (max 14 дней).

## Жёсткие ограничения

1. **Только OWL DL**: MVP поддерживает только OWL DL (не Full)
2. **Без встроенного Reasoner**: Pellets/HermiT не реализуются, только API для внешних
3. **Без мобильного редактирования**: Только web UI
4. **Без хранения бинарных BLOB**: Встраивание файлов в онтологию не поддерживается
5. **Без SWRL UI**: SWRL правила только как экспорт/импорт текста
6. **Лицензия MIT**: VEDO Core выпускается под лицензией MIT.
7. **Граница лицензирования стороннего runtime**: MIT распространяется на first-party код VEDO Core; сторонние runtime-компоненты сохраняют собственные лицензии. Базовая MIT-поставка не должна требовать Neo4j Enterprise-only features; Neo4j Enterprise cluster/HA/backup используются только как BYOL/customer-supplied или managed subscription.
8. **`vedo-cli` как административная граница**: `vedo-cli` является обязательной административной утилитой экосистемы VEDO Core для backup/restore, миграций, air-gapped подготовки, экспорта/импорта онтологий и диагностики инцидентов. Веб-интерфейс не заменяет `vedo-cli` для privileged, automated и blind-environment операций.

## Ограничения API

- REST endpoints требуют JWT-токен (Keycloak)
- gRPC для внутреннего взаимодействия между сервисами
- WebSocket — опционально (post-MVP) для уведомлений через API Gateway
- GraphQL endpoint `/graphql` является основным API для frontend-навигации по графу онтологии.
- GraphQL navigation API должен поддерживать cursor pagination для всех списков, включая `children`, `parents`, `incomingEdges`, `outgoingEdges` и `propertyValues`.
- GraphQL navigation API должен возвращать динамические свойства онтологии через контейнеры `propertyValues` и связи через `outgoingEdges`/`incomingEdges`, а не через заранее сгенерированные поля для каждого пользовательского свойства.
- Все public endpoints требуют authentication
- **OAuth Providers**: Поддержка авторизации через российские OAuth провайдеры (ЕСИА/Госуслуги, Яндекс, VK) и Google
- SPARQL endpoint `/api/v1/sparql` принимает только read-only запросы SELECT/ASK; INSERT, DELETE, UPDATE и другие модифицирующие формы блокируются до выполнения.
- SPARQL endpoint использует параметризованные запросы или строго контролируемое экранирование значений, если библиотека не поддерживает полноценную параметризацию.
- SELECT-запросы без явного `LIMIT` отклоняются, кроме согласованных исключений.
- Для SPARQL endpoint действуют timeout, result cap, rate limiting, query complexity gate и audit logging.

## Требования к выполнению GraphQL-запросов навигации

- Timeout выполнения GraphQL-навигационного запроса по умолчанию: 10 секунд, если контракт или окружение не задаёт более строгий лимит.
- Максимальный объём ответа по умолчанию: 1000 узлов или связей на один запрос.
- Максимальное значение `first` для connection-полей по умолчанию: 100 элементов.
- Depth limit по умолчанию: 5 уровней вложенности для навигационных запросов.
- Query complexity threshold по умолчанию: 1000 баллов на основе весов полей, глубины вложенности и requested page size.
- Rate limit по умолчанию: 60 GraphQL-навигационных запросов в минуту на пользователя.
- Запросы, превышающие depth limit, complexity threshold, timeout или result cap, отклоняются с понятной ошибкой и рекомендацией сузить запрос.
- Все GraphQL-запросы логируются с actor, role, timestamp, trace_id, operation name, redacted variables summary, result count, duration и решением security checks.

## Требования к выполнению SPARQL-запросов

- Timeout выполнения SPARQL-запроса по умолчанию: 30 секунд, если контракт или окружение не задаёт более строгий лимит.
- Максимальный объём ответа по умолчанию: 10 000 записей.
- Rate limit по умолчанию: 10 SPARQL-запросов в минуту на пользователя.
- Query complexity threshold по умолчанию: 1000 баллов на основе AST запроса.
- Все SPARQL-запросы логируются с actor, role, timestamp, trace_id, redacted query summary, result count, duration и решением security checks.
- Логи SPARQL-запросов не должны содержать секреты, токены и персональные данные.

## Требования к наблюдаемости

- Базовая поставка VEDO Core обязана включать OpenTelemetry Collector, Prometheus, Grafana, Grafana Loki и Grafana Tempo.
- OpenTelemetry instrumentation обязательна для всех сервисов.
- RED method metrics (Rate, Errors, Duration) обязательны для всех публичных endpoints и внутренних service calls.
- Все сервисы экспортируют `vedo_*` метрики в Prometheus-compatible формате.
- Trace spans обязательны для операций с Neo4j, PostgreSQL, Redis, RabbitMQ, HTTP и gRPC вызовов.
- Единый overview dashboard обязателен для базового мониторинга health, latency, error rate, saturation, логов и трейсов.
- Loki используется как backend логов по умолчанию; Tempo используется как backend трейсов по умолчанию.
- Альтернативные backend observability допустимы только как опция заказчика через OpenTelemetry exporters или Prometheus remote write и не заменяют поставку по умолчанию.
- Поддержка Datadog, New Relic, Elastic Stack/ELK, Jaeger или Victoria Metrics вне базового стека является зоной ответственности заказчика либо отдельной платной услугой.
- Retention по умолчанию: метрики 30 дней, логи 30 дней, трейсы 7 дней.
- `vedo-cli diagnose trace --id <trace_id>` должен работать как единая точка входа в OpenTelemetry stack: Tempo для trace/span, Loki для logs, Prometheus для metrics.
- Smart diagnostics должен локализовать проблемный span, показать связанные логи и метрики за интервал инцидента и сформировать рекомендации для инженера поддержки; LLM-анализ является опциональным режимом и не должен быть обязательным для air-gapped runtime.

## Требования к backup, restore и миграциям

- Full backup включает TBox в canonical Turtle, ABox как Neo4j binary dump, Version Store через `pg_dump -Fc`, WAL/incremental logs для PITR и LFS-объекты в S3/MinIO-compatible storage.
- Backup считается успешным только после `vedo-cli backup verify` или эквивалентной автоматической проверки целостности.
- Ежедневный automated backup должен настраиваться через `vedo-cli` или сгенерированный `vedo-cli` schedule/job manifest.
- Restore должен выполняться по RTO-протоколу через `vedo-cli restore` и фиксировать audit trail.
- Neo4j и PostgreSQL migrations должны быть идемпотентными, иметь pre-migration backup и поддерживать rollback одной командой через `vedo-cli migrate rollback`.

## Решения по развёртыванию

- ~~Требования к geographically distributed deployment?~~ — один код, разные регионы; РФ включена в MVP, ЕС и США поддерживаются в P2, другие регионы по запросу
- ~~Бюджет на инфраструктуру Grafana stack?~~ — минимальный

## Резидентность данных

- РФ: персональные данные должны храниться на серверах на территории РФ; рекомендуемые провайдеры Yandex Cloud, VK Cloud, Selectel.
- ЕС: персональные данные должны храниться в ЕС; рекомендуемые регионы AWS Frankfurt, Azure Germany или Google Cloud Frankfurt.
- США: допускается deployment в AWS us-east-1 или Google Cloud us-central1; HIPAA и SOC 2 Type II применяются по типу клиента.
- ПДн пользователей и логи не синхронизируются между регионами.
- Обезличенные TBox модели могут синхронизироваться между регионами по согласованию.

## Развёртывание в изолированном контуре

- VEDO Core должен поддерживать on-premise работу без доступа в интернет.
- Лицензия MIT не требует online activation, trial checks или license server.
- Runtime services не должны требовать outbound internet access.
- Air-gap installation использует local registry: Harbor, Nexus или аналог.
- Helm charts должны устанавливаться из local directory или local chart mirror.
- Offline mode (`VEDO_OFFLINE_MODE=true`) отключает telemetry, version checks и внешние imports по умолчанию.
- PagerDuty и Slack недоступны в air-gap; alerting выполняется через SMTP внутри security perimeter.
- Backup в air-gap использует local storage или MinIO/S3-compatible storage внутри контура.
- Документация должна быть доступна локально для air-gapped customers.

## Устойчивое развитие и энергоэффективность

- Sustainability requirements не блокируют MVP, если customer contract явно не требует обратного.
- Energy per API request target: < 0.5 J/request.
- Carbon per active user target: < 100 g CO2e/month.
- Idle energy consumption target: < 30% of peak consumption.
- Average cluster CPU utilization target: > 40%.
- Для enterprise deployments могут поставляться Kepler/Scaphandre metrics и Grafana dashboards.
- Enterprise customers могут запрашивать monthly SCI/carbon footprint report.
- Green cloud region selection рекомендуется, если не конфликтует с data residency или customer policy.

## Требования CI/CD

- GitLab CI является CI/CD платформой по умолчанию для VEDO Core.
- Репозиторий должен поставляться с `.gitlab-ci.yml`.
- Push в любую ветку должен запускать linting для Rust, Go, Python и TypeScript.
- Push в любую ветку должен запускать unit tests для Rust, Go, Python и TypeScript.
- Merge Request в `main` должен запускать API integration tests.
- Push в `main` должен собирать Docker images.
- Push в `main` должен собирать документацию через Antora.
- Deploy в dev выполняется автоматически после успешного CI в `main`.
- Deploy в staging выполняется вручную.
- Deploy в production выполняется вручную с подтверждением.
- Production rollback выполняется вручную и должен поддерживать откат до 5 версий.
- Antora documentation публикуется из CI artifacts `public/`.

## Требования к стратегии развёртывания

- Rolling update является deployment strategy по умолчанию для dev, staging и production minor/patch updates.
- Production major updates должны использовать blue-green deployment при наличии достаточных ресурсов.
- Если blue-green невозможен из-за ресурсов, major update должен выполняться через documented maintenance window и rollback plan.
- Canary release является опциональным и требует service mesh: Istio, Linkerd или аналог.
- Canary release рекомендуется только для крупных installations или A/B testing.
- Helm chart должен поддерживать rolling update по умолчанию.
- Helm chart должен предоставлять настройки `bluegreen.enabled` и `canary.enabled`.
- Production traffic switch и rollback выполняются вручную.

## Требования к конфигурации окружений

- Git repository является source of truth для non-secret конфигурации окружений.
- `helm-chart/values.yaml` содержит default values и не должен редактироваться напрямую под конкретную среду.
- Environment-specific overrides должны храниться в `k8s/dev/values.yaml`, `k8s/staging/values.yaml`, `k8s/prod/values.yaml`.
- GitLab CI deploy jobs должны применять соответствующий values file через `helm upgrade --values`.
- Secrets запрещено хранить в Git и plain values files.
- Secrets должны храниться в masked GitLab CI/CD Variables, Kubernetes Secrets или external secret manager.
- Production по возможности должен использовать external secret operator с AWS Secrets Manager, HashiCorp Vault или аналогом.
- Production configuration должна включать monitoring, SSL и backups.
- Configuration changes должны проходить review и CI/CD как code changes.

Подробнее: `human/artifacts/requirements/REQ-NFR.PERF.performance.md`

## Стратегия развёртывания

**Платформа:** Облако (дешёвый вариант на первом этапе)
**Кандидаты:**
- Hetzner Cloud (Германия) — €4-20/мес за VPS
- DigitalOcean — $4-48/мес
- Яндекс.Облако / VK Cloud (для российских клиентов)

**Стратегия минимизации стоимости:**
1. Single-node deployment на начальном этапе
2. Все сервисы в одном контейнере или минимальное количество
3. PostgreSQL + Redis на том же хосте (не managed services)
4. OpenTelemetry Collector sidecar вместо managed observability
5. Loki/Prometheus в single-node mode

**Следующий этап (масштабирование):**
- Kubernetes (K3s на cheap VPS)
- Managed PostgreSQL ( RDS, Cloud SQL)
- Managed Redis (ElastiCache, MemoryDB)
- Grafana Cloud или self-hosted
