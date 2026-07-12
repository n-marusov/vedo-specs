# VEDO CLI — Спецификация требований к административной утилите

| Атрибут | Значение |
|---------|----------|
| **ID** | REQ-FUN.INFRA.vedo-cli-specification |
| **Уровень** | FUN |
| **Атрибут качества** | Implementation |
| **Приоритет** | P0 |
| **Статус** | УТВЕРЖДЕНО |
| **Источник** | — |
| **Критерии приёмки** | — |

---

## Обзор

`vedo-cli` — единая административная утилита командной строки для экосистемы VEDO Core. Консолидирует операции, которые иначе распадаются на `kubectl`, `helm`, `neo4j-admin`, `psql`, S3-клиенты и ручные Grafana/Loki/Tempo/Prometheus запросы.

`vedo-cli` не заменяет web UI и public API. Зона ответственности ограничена операциями, требующими автоматизации, повышенных привилегий или работы в air-gapped окружениях.

## Бизнес-контекст

### Бизнес-цель

`vedo-cli` является ключевым инструментом достижения **G5 (Эксплуатационная надёжность и Enterprise-поддержка)**: P0-инцидент локализуется за ≤ 10 мин, реакция на P0-тикет — ≤ 15 мин (24/7), restore из backup — по RTO-протоколу.

### Решаемые проблемы

- **P7** (разрозненные инструменты администрирования): единый CLI вместо `kubectl`, `helm`, `neo4j-admin`, `psql`, S3-клиентов, Grafana/Loki/Tempo/Prometheus.
- **P8** (ночной P0 без поддержки): `vedo-cli diagnose` как единая точка входа в диагностику; возможность включения в runbook для L1-инженера.

### Соответствие функциям

- **F9** (Администрирование через `vedo-cli`): backup/restore, миграции, air-gap, диагностика.
- **F12** (Поддержка и сбор обратной связи): support-команды (tenant-info, audit-trail, emergency-access) и управление тикетами.
- **F3** (Управление версиями): управление Merge Request'ами, ветками, коммитами.

### Нормативные ограничения

Из `human/artifacts/requirements/REQ-CON.CROSS.constraints.md`:
- **admin_operations_via_vedo_cli**: Прямой административный доступ к Neo4j/PostgreSQL/S3/MinIO в production допускается только через `vedo-cli` или согласованный break-glass процесс.
- **admin_audit_required**: Все действия `vedo-cli` логируются с actor, role, command, target environment, trace_id/correlation_id, result и redacted input summary.
- **`vedo-cli` как административная граница**: `vedo-cli` является обязательной административной утилитой экосистемы VEDO Core. Веб-интерфейс не заменяет `vedo-cli` для privileged, automated и blind-environment операций.

### Показатели эффективности

| Метрика | Значение | Источник |
|---------|----------|----------|
| Time-to-Diagnose P0 | ≤ 10 минут через `vedo-cli diagnose` | `constraints.md` |
| Time-to-Diagnose P1 | ≤ 30 минут через `vedo-cli diagnose` | `constraints.md` |
| Верификация backup | Backup считается успешным только после `vedo-cli backup verify` | `constraints.md` |
| Migrations rollback | Одной командой через `vedo-cli migrate rollback` | `constraints.md` |

---

## Пользователи

| Профиль | Роль | Сценарии |
|---------|------|----------|
| **DevOps-инженер** | Эксплуатация | backup/restore, миграции, air-gap подготовка, развёртывание, SCA/SBOM |
| **Инженер поддержки (SRE)** | Диагностика | diagnose trace_id, региональная диагностика, support-команды |
| **Security Lead** | Безопасность | emergency access, compliance evidence, escalation matrix, SCA |
| **Администратор онтологий** | Данные | экспорт/импорт/сравнение онтологий, перенос между окружениями |
| **Owner группы/онтологии** | Управление | управление членством, передача владения |

---

## Архитектура

### Принципиальная схема

```
vedo-cli (binary)
  │
  ├──→ Keycloak (OIDC ROPG+TOTP / client_credentials)
  │     └──→ Получение access_token, refresh_token
  │
  ├──→ API Gateway (REST admin endpoints)
  │     ├──→ Ontology Service (Rust)
  │     ├──→ Versioning Service (Rust)
  │     └──→ Auth Service (Go / Keycloak — верификация токенов)
  │     └──→ Ticket API (Go / Ticket Management)
  │
  ├──→ Neo4j (прямые операции: backup, migrate)
  ├──→ PostgreSQL (прямые операции: backup, migrate)
  ├──→ S3/MinIO (backup storage)
  ├──→ Tempo (query traces)
  ├──→ Loki (query logs)
  ├──→ Prometheus (query metrics)
  │
  ├──→ Support DB (отдельный PostgreSQL: tenant-info, audit-trail, escalation-matrix, tickets)
  ├──→ WORM S3 (Object Lock: compliance-evidence, audit archive)
  └──→ Vault / KMS (emergency-access keys)
```

### Тип исполнения

`vedo-cli` — бинарный CLI-инструмент, не Docker-сервис. Предназначен для запуска на рабочей станции оператора, в CI/CD пайплайнах и в air-gapped окружениях без доступа к registry.

### Взаимодействие с сервисами

- Ключевые административные операции (создание access point, привязка storage, установка квот, настройка backup policy) выполняются через REST API Gateway (`POST /api/v1/admin/*`).
- Прямые операции с БД (backup, migrate) выполняются напрямую без промежуточных сервисов.

## Функциональные требования

### Команды

#### REQ-VC-BACKUP-001: Backup и Restore

| Команда | Описание | Guardrail |
|---------|----------|-----------|
| `vedo-cli backup create --full` | Полный backup Neo4j + PostgreSQL, загрузка в S3/MinIO, верификация | — |
| `vedo-cli backup verify` | Проверка целостности backup-пакета | — |
| `vedo-cli backup delete` | Удаление backup | G3 |
| `vedo-cli backup schedule` | Настройка расписания backup | — |
| `vedo-cli restore --id <backup_id>` | Восстановление из backup | G3 |
| `vedo-cli backup purge-request --tenant <id> --jurisdiction <jurisdiction>` | Запрос на удаление backup тенанта по юрисдикции | — |

**Детали:**
- Full backup включает TBox в canonical Turtle, ABox как Neo4j binary dump, Version Store через `pg_dump -Fc`, WAL/incremental logs для PITR и LFS-объекты в S3/MinIO-compatible storage.
- Backup считается успешным только после `vedo-cli backup verify`.
- Ежедневный automated backup настраивается через `vedo-cli` или сгенерированный `vedo-cli` schedule/job manifest.
- Restore выполняется по RTO-протоколу и фиксирует audit trail.

*Источники: `docs/vedo-cli.md`, `constraints.md`, `sequences.md UC-admin.backup.manage-backup-and-restore-via-cli`*

#### REQ-VC-MIGRATE-002: Миграции

| Команда | Описание | Guardrail |
|---------|----------|-----------|
| `vedo-cli migrate plan` | Чтение состояния миграций, формирование плана | — |
| `vedo-cli migrate apply` | Применение миграций PostgreSQL + Neo4j с проверкой pre-migration backup | G3 |
| `vedo-cli migrate rollback` | Откат миграции из pre-migration backup | G3 |

**Детали:**
- Neo4j и PostgreSQL migrations должны быть идемпотентными.
- Каждая миграция создаёт pre-migration backup.
- Rollback выполняется одной командой.
- После apply выполняются integrity checks (counts, history, SPARQL spot checks).

*Источники: `docs/vedo-cli.md`, `constraints.md`, `sequences.md UC-admin.migration.apply-migrations-with-rollback-via-cli`*

#### REQ-VC-DIAGNOSE-003: Диагностика, Emergency и Кэш

| Команда | Описание | Guardrail |
|---------|----------|-----------|
| `vedo-cli diagnose trace --id <trace_id>` | Поиск trace в Tempo, логов в Loki, метрик в Prometheus | — |
| `vedo-cli diagnose` | Общая диагностика состояния системы | — |
| `vedo-cli diagnose region --region <id>` | Диагностика региона (health, replication lag, ошибки) | — |
| `vedo-cli cache warmup` | Разогрев кэша Redis в регионе восстановления после failover | — |
| `vedo-cli emergency readonly` | Kill Switch — перевод системы в режим «только чтение» | — |
| `vedo-cli emergency clear` | Снятие emergency-режима | — |

**Детали диагностики:**
- `vedo-cli diagnose trace --id <trace_id>` запрашивает trace из Tempo, извлекает ошибочный `span_id` и связанные span attributes, затем запрашивает логи из Loki и метрики из Prometheus за соответствующий интервал.
- LLM-режим опционален (флаг `--llm`), в air-gapped окружениях отключён по умолчанию.
- Все сервисы обязаны корректно инструментировать traces, logs и metrics с correlation IDs.

**Детали Emergency:**
- **L1: read-only mode** — `vedo-cli emergency readonly` переводит API Gateway в режим отклонения всех mutating запросов (HTTP 503).
- **L2: service-level kill** — `vedo-cli emergency kill --service <name>` отключает конкретный сервис.
- Обратный переход — только после `vedo-cli emergency clear`.
- `vedo-cli emergency readonly` выполняется без прохождения стандартной очереди запросов (priority path).

*Источники: `docs/vedo-cli.md`, `critical-alerts.md`, `edge-region-failover.md`*

#### REQ-VC-DECOMMISSION-004: Decommission и Data Lifecycle

| Команда | Описание | Guardrail |
|---------|----------|-----------|
| `vedo-cli decommission plan --tenant <id>` | Dry-run план вывода тенанта (без мутации состояния) | — |
| `vedo-cli decommission export` | Экспорт данных тенанта | — |
| `vedo-cli decommission verify --package <dir>` | Верификация экспортированного пакета | — |
| `vedo-cli decommission --purge-data` | Безвозвратное удаление данных тенанта | G4 |
| `vedo-cli decommission --verify-purge` | Проверка отсутствия остаточных данных | — |
| `vedo-cli account close --account-id <id> --jurisdiction <gdpr\|ru_152fz>` | Закрытие аккаунта по юрисдикции | G3 |

**Детали:**
- Decommission — идемпотентный процесс с машиной состояний (not_started → exporting → export_verified → restore_verified → pending_retention_expiry → crypto_erased → completed).
- `--purge-data` требует проверки всех данных Neo4j, WAL PostgreSQL, RDB/AOF Redis, tenant-специфичных бакетов в S3/MinIO, кэшей API Gateway и backup-копий.
- `--verify-purge` проверяет отсутствие остаточных данных во всех хранилищах.

**Форматы экспорта:**
| Формат | Команда | Назначение |
|--------|---------|------------|
| Turtle (TTL) | `vedo-cli export --format turtle` | TBox, ABox для миграции |
| JSON-LD | `vedo-cli export --format json-ld` | ABox для web-интеграции |
| Git | `vedo-cli export --format git` | Version Store |
| JSON Lines | `vedo-cli export --format jsonl --type commits\|audit` | Аналитика, SIEM |
| YAML/JSON | `vedo-cli export --format json --type rbac\|config` | Конфигурация |
| CSV | `vedo-cli export --format csv` | Аналитика, Excel |
| Parquet | `vedo-cli export --format parquet` | Data warehouse |
| SQL INSERT | `vedo-cli export --format sql` | Восстановление в реляционную БД |

*Источники: `docs/vedo-cli.md`, `decommission-export.md`, `account-closure-retention.md`*

#### REQ-VC-ONTOLOGY-005: Перенос и администрирование онтологий

Набор команд для административного переноса онтологий между окружениями (например, production → staging) и управления онтологиями. Не является пользовательским импортом/экспортом файлов (F8).

| Команда | Описание | Guardrail |
|---------|----------|-----------|
| `vedo-cli ontology export --env <env> --canonical` | Экспорт TBox онтологии в canonical Turtle для переноса в другое окружение | — |
| `vedo-cli ontology import --env <env> <file>` | Импорт canonical Turtle в целевое окружение | — |
| `vedo-cli ontology diff <env1> <env2>` | Semantic diff между окружениями | — |
| `vedo-cli ontology delete` | Удаление онтологии | G3 |
| `vedo-cli ontology restore --ontology-id <id>` | Восстановление онтологии из backup | — |
| `vedo-cli ontology transfer-owner --ontology-id <id> --new-owner <user>` | Передача владения онтологией | — |

*Источники: `docs/vedo-cli.md`, `sequences.md UC-admin.migration.transfer-and-diff-ontologies-via-cli`*

#### REQ-VC-TENANT-006: Tenant Management

| Команда | Описание | Guardrail |
|---------|----------|-----------|
| `vedo-cli tenant seed --profile canonical` | Развёртывание эталонного профиля нагрузки (1M аксиом) | — |
| `vedo-cli tenant seed --force` | Принудительный seed | G2 |
| `vedo-cli tenant delete` | Удаление тенанта | G4 |
| `vedo-cli tenant restore --tenant-id <id>` | Восстановление тенанта | — |
| `vedo-cli tenant rename --tenant-id <id> --name <name>` | Переименование тенанта | — |

*Источники: `docs/vedo-cli.md`*

#### REQ-VC-AIRGAP-007: Air-Gap подготовка

| Команда | Описание | Guardrail |
|---------|----------|-----------|
| `vedo-cli airgap prepare` | Сбор offline-пакета (образы, Helm-чарты, docs, checksums) | — |
| `vedo-cli airgap prepare --advisory-bundle <file>` | Подготовка SCA advisory bundle для offline-сканирования | — |
| `vedo-cli airgap verify package` | Проверка целостности offline-пакета | — |
| `vedo-cli airgap import-advisories --bundle <file>` | Импорт advisory баз в изолированном контуре | — |
| `vedo-cli airgap check-advisories --max-age-hours 48` | Проверка свежести advisory баз | — |

**Детали:**
- `vedo-cli airgap prepare` собирает Docker images, Helm-чарты, документацию, конфигурацию и контрольные суммы.
- `vedo-cli airgap verify` проверяет целостность собранного пакета.
- В air-gapped режиме (`VEDO_OFFLINE_MODE=true`) отключаются телеметрия, version checks и внешние imports.
- SCA-сканирование в air-gapped среде использует локальные advisory базы.

*Источники: `docs/vedo-cli.md`, `air-gapped-deployment.md`, `sca-sbom-gating.md`*

#### REQ-VC-SCA-008: SCA / SBOM (Supply Chain Security)

| Команда | Описание | Guardrail |
|---------|----------|-----------|
| `vedo-cli sca scan <target>` | SCA-сканирование зависимостей (RustSec, Trivy, Safety, Govulncheck) | — |
| `vedo-cli sca sbom generate <target>` | Генерация SBOM в SPDX 2.3 / CycloneDX 1.5 | — |

*Источники: `docs/vedo-cli.md`, `sca-sbom-gating.md`*

#### REQ-VC-VERSION-009: Version Control (Merge Requests)

| Команда | Описание | Guardrail |
|---------|----------|-----------|
| `vedo-cli branch delete` | Удаление ветки | G1 |
| `vedo-cli commit revert` | Откат коммита | G1 |
| `vedo-cli mr create --source <branch> --target main` | Создание Merge Request | — |
| `vedo-cli mr list --status open` | Список Merge Request'ов по статусу | — |
| `vedo-cli mr review --id <id> --approve` | Ревью и утверждение Merge Request | — |
| `vedo-cli mr merge --id <id>` | Слияние одобренного Merge Request | G1 |

*Источники: `docs/vedo-cli.md`*

#### REQ-VC-SUPPORT-010: Support Metadata

| Команда | Описание | Guardrail |
|---------|----------|-----------|
| `vedo-cli support tenant-info <id>` | Полная информация о tenant (контакты, SLA, deployment config) | — |
| `vedo-cli support tenant-info --search <name/email>` | Поиск tenant по названию или email | — |
| `vedo-cli support audit-trail <id>` | Audit trail tenant (все события) | — |
| `vedo-cli support audit-trail <id> --since <date>` | Фильтр audit trail по дате | — |
| `vedo-cli support audit-trail <id> --type <event_type>` | Фильтр audit trail по типу события | — |
| `vedo-cli support list-backups <id>` | Список backup tenant | — |
| `vedo-cli support list-backups <id> --type worm` | Только WORM archive | — |
| `vedo-cli support list-backups <id> --status verified` | Только верифицированные backup | — |
| `vedo-cli support emergency-access request <id> --reason "<reason>"` | Запрос break-glass ключа | G3 |
| `vedo-cli support emergency-access list <id>` | История выданных emergency-ключей | — |
| `vedo-cli support emergency-access revoke <key_id>` | Отзыв ключа emergency-доступа | G3 |
| `vedo-cli support compliance-evidence list <id>` | Список compliance evidence tenant | — |
| `vedo-cli support compliance-evidence upload <id> --type dpa --file <file>` | Загрузка compliance evidence (DPA, SOC2, ISO27001) | G3 |
| `vedo-cli support escalation-matrix` | Текущая матрица эскалации (SLA tiers, response times) | — |
| `vedo-cli support escalation-matrix --tenant <id>` | Матрица эскалации с учётом SLA tenant | — |
| `vedo-cli support health` | Статус всех компонентов support storage | — |

**Детали:**
- Команды `vedo-cli support` работают полностью offline в air-gapped окружениях, обращаясь к локальной Support DB, MinIO и Vault.
- Support DB — отдельный PostgreSQL-инстанс, изолированный от основных сервисов VEDO Core.

*Источники: `docs/vedo-cli.md`, `support-metadata-isolation.md`*

#### REQ-VC-TICKET-011: Управление тикетами (CRUD)

| Команда | Описание | Guardrail |
|---------|----------|-----------|
| `vedo-cli ticket create --title "<title>" --description "<text>" --category <category> --severity <level>` | Создать тикет вручную через CLI | — |
| `vedo-cli ticket list --status <status> --category <category> --source <manual\|telemetry>` | Список тикетов с фильтрацией | — |
| `vedo-cli ticket get --id <ticket_id>` | Просмотр карточки тикета и истории изменений | — |
| `vedo-cli ticket update --id <ticket_id> --priority <p0\|p1\|p2\|p3> --assignee <user>` | Обновить поля тикета | — |
| `vedo-cli ticket comment --id <ticket_id> --text "<comment>"` | Добавить комментарий в тикет | — |
| `vedo-cli ticket close --id <ticket_id> --resolution "<text>"` | Закрыть тикет | — |
| `vedo-cli ticket reopen --id <ticket_id> --reason "<text>"` | Переоткрыть тикет | — |
| `vedo-cli ticket delete --id <ticket_id>` | Удалить тикет | G3 |

**Детали:**
- Все команды `vedo-cli ticket *` работают с тем же бэкендом и той же моделью жизненного цикла, что и UI.
- В историю тикета записывается канал операции `channel=cli`.
- Для `delete` обязательны MFA, env guard и typed confirmation.
- Команды поддерживают `--format json` для CI/CD и интеграционных скриптов.

*Источники: `docs/vedo-cli.md`, `ticket-management-system.md` (`T-43`-`T-47`)*

#### REQ-VC-SECURITY-012: Emergency Security Policy

| Команда | Описание | Guardrail |
|---------|----------|-----------|
| `vedo-cli security policy disable --scope global --ticket <id>` | Экстренное отключение политики безопасности | G3 |

*Источник: `emergency-security-policy-disable.md`*

---

## Аутентификация и авторизация

### REQ-VC-AUTH-001: Принципы

- Все привилегированные операции требуют аутентификации через Keycloak (OIDC). Решение зафиксировано в `ADR-DES.SECURITY.cli-mfa-strategy`.
- **Интерактивный режим:** OIDC Resource Owner Password Grant (ROPG) + TOTP claim. `access_token` (TTL: 1 час), `refresh_token` кэшируется в `~/.vedo/config.yaml` (зашифрован).
- **Неинтерактивный режим (CI/CD):** Service account с `client_credentials` grant. Допустимы операции категорий C/D при явном ограниченном scope. Операции A/B через service account запрещены.
- **MFA-политика по категориям:**
  - **A/B:** MFA обязателен при каждом вызове, не кэшируется (даже при живом `access_token`).
  - **C:** MFA запрашивается 1 раз за время жизни refresh_token.
  - **D:** MFA не требуется.
- Для service-account MFA-челлендж не применяется; безопасность обеспечивается ограниченным scope токена, коротким TTL, аудитом и запретом A/B.
- Права проверяются через AuthDecision — только роль admin допускает admin-операции.
- Операции скоупированы по тенанту.
- Emergency Admin (L1-L3) — отдельная учётная запись вне Keycloak с bcrypt/argon2 паролем по схеме Шамира (M из N).

### REQ-VC-AUTH-003: Service-account token scope (дополнение)

Токен service-account должен иметь явный список разрешенных операций (allow-list).

Пример конфигурации токена:

```yaml
# ~/.vedo/service-account-token.yaml
token:
  type: "service_account"
  client_id: "ci-cd-pipeline-123"
  allowed_operations:
    - "ontology:delete"
    - "branch:delete"
    - "ontology:export"
    - "backup:create"
    - "diagnose:trace"
  forbidden_operations:
    - "tenant:delete"
    - "backup:delete"
    - "migration:rollback"
    - "restore:production"
  expires_in_seconds: 3600
  allowed_ips:
    - "10.0.0.0/8"
    - "172.16.0.0/12"
```

Принцип минимальных привилегий: токен получает доступ только к операциям, которые реально нужны конкретной автоматизации; выдача полного доступа ко всем операциям C/D запрещена.

Рекомендации по TTL:

| Сценарий | Рекомендуемый TTL | Обоснование |
|----------|-------------------|-------------|
| CI/CD пайплайн | 15-30 минут | Обычно короче времени выполнения пайплайна |
| Cron-задача (ежечасная) | 1 час | Покрывает джиттер и время выполнения |
| Скриптовый неинтерактивный запуск | 1 час | Компромисс между удобством и риском |

Аудит использования service-account токена:

```json
{
  "event": "service_account_operation",
  "timestamp": "2026-05-23T10:00:00Z",
  "token_id": "sa-123",
  "client_ip": "10.0.0.42",
  "operation": "ontology:delete",
  "tenant_id": "tenant-456",
  "success": true,
  "correlation_id": "ci-pipeline-789"
}
```

Согласованность с ADR: данный раздел детализирует `ADR-DES.SECURITY.mfa-critical-ops-mandate` для подсистемы CLI auth и является авторитетным при интерпретации service-account правил в CLI.

### REQ-VC-AUTH-002: Destructive Command Guardrails

| Уровень | Требования | Команды |
|---------|------------|---------|
| G1 | typed confirmation | branch delete, commit revert |
| G2 | env guard + typed confirmation | tenant seed --force |
| G3 | MFA + env guard + typed confirmation | ontology delete, account close, backup delete, migrate rollback, migrate apply, decommission purge, ticket delete, support emergency-access request, support emergency-access revoke, support compliance-evidence upload |
| G4 | cool-down delay + full backup verification + всё из G3 | tenant delete, decommission --purge-data |

Peer approval для разрушительных операций: `vedo-cli --require-peer-approval`, подтверждение через `vedo-cli approval approve <request-id>`.

Примечание: для service-account peer approval и MFA не заменяют ограничение scope; операции категорий A/B остаются недоступными независимо от guardrail-флагов.

---

## Выходные форматы

### REQ-VC-OUTPUT-001: Форматы

- `--format json` — machine-readable вывод для automation и CI/CD.
- `--format human` (по умолчанию) — человекочитаемый вывод для интерактивной работы.

Все команды должны возвращать:
- Стандартные exit codes (0 = успех, 1 = ошибка, 2 = неверные аргументы).
- Сообщения об ошибках в stderr.
- JSON-вывод с полем `status` (ok/error) и `data`/`error`.

---

## Версионирование

### REQ-VC-VERSION-001: Политика версий

- `vedo-cli` версионируется вместе с VEDO Core release.
- Совместимость команд с версиями серверных служб поддерживается на уровне мажорной версии.

### REQ-VC-DISTRIB-001: Распространение

- `vedo-cli` поставляется как single binary под каждую целевую платформу.
- В каждом релизе публикуются ссылки на бинарные артефакты для всех поддерживаемых комбинаций ОС и архитектуры.
- Поддерживаемые платформы: linux/amd64, linux/arm64, darwin/amd64, darwin/arm64, windows/amd64.
- Установка через package manager (Homebrew, Scoop, apt) — опционально, во вторую очередь.
- Бинарный артефакт не требует внешних зависимостей или рантайма (Go single binary).

---

## Аудит

### REQ-VC-AUDIT-001: Требования к логированию

Все действия `vedo-cli` логируются с обязательными полями:
- actor (кто выполнил)
- role (роль пользователя)
- command (полная команда с аргументами)
- target environment
- trace_id / correlation_id
- result (успех/ошибка)
- redacted input summary (PII/секреты маскируются)

---

## Требования к управлению секретами

### REQ-VC-SECRETS-001: Хранение credentials для backup storage

`vedo-cli` для выполнения backup/restore операций требуется доступ к S3/MinIO-compatible storage. Учётные данные (access key, secret key, endpoint) не должны храниться в конфигурационных файлах или передаваться как plain-text аргументы команд.

**Поддерживаемые источники credentials (в порядке приоритета):**

| Источник | Механизм | Сценарий использования |
|----------|----------|----------------------|
| HashiCorp Vault | `VAULT_ADDR` + `VAULT_TOKEN`, путь `secret/vedo/backup/<env>` | on-premise / air-gapped, где уже используется Vault |
| AWS Secrets Manager | IAM role или static key, имя секрета `vedo/backup/<env>` | SaaS / cloud deployment |
| Переменные окружения | `VEDO_BACKUP_S3_ACCESS_KEY`, `VEDO_BACKUP_S3_SECRET_KEY` | CI/CD, dev-окружения, fallback |
| Plain-text конфиг | Файл `backup-credentials.env` за пределами репозитория | Аварийный fallback только для air-gapped без Vault |

**Правила разрешения:**
1. Если доступен Vault (`VAULT_ADDR` задан) — приоритет Vault.
2. Если `AWS_SECRETS_MANAGER_REGION` задан — приоритет AWS Secrets Manager.
3. Если заданы переменные окружения `VEDO_BACKUP_S3_*` — использовать их.
4. Если ничего не задано — ошибка с сообщением `credentials source not configured, see --help`.

**Требования:**
- Секреты никогда не выводятся в stdout/stderr и не логируются (redacted как `***`).
- Команда `vedo-cli backup create` проверяет доступность storage с переданными credentials перед началом backup.
- При недоступности storage команда завершается с ошибкой до начала dump БД.

### REQ-VC-OBS-001: Интеграция с observability stack

- `vedo-cli diagnose trace --id <trace_id>` должен работать как единая точка входа в OpenTelemetry stack:
  - Tempo для trace/span
  - Loki для logs
  - Prometheus для metrics
- Smart diagnostics должен локализовать проблемный span, показать связанные логи и метрики за интервал инцидента и сформировать рекомендации.
- LLM-анализ является опциональным режимом и не должен быть обязательным для air-gapped runtime.

---

## Связанные артефакты

| Артефакт | Содержание |
|----------|------------|
| `docs/vedo-cli.md` | Полное описание команд, архитектуры, guardrails |
| `constraints.md` | Нормативные ограничения (admin_operations_via_vedo_cli, admin_audit) |
| `sequences.md` (UC-admin.backup.manage-backup-and-restore-via-cli, UC-admin.migration.apply-migrations-with-rollback-via-cli, UC-admin.airgap.prepare-air-gapped-package-via-cli, UC-diagnostics.trace.diagnose-incident-by-trace-id, UC-admin.migration.transfer-and-diff-ontologies-via-cli) | Sequence-диаграммы ключевых сценариев |
| `support-metadata-isolation.md` | Support DB и vedo-cli support команды |
| `decommission-export.md` | Форматы экспорта данных |
| `critical-alerts.md` | Emergency readonly/kill |
| `air-gapped-deployment.md` | Air-gap подготовка |
| `sca-sbom-gating.md` | SCA-сканирование |
| `edge-region-failover.md` | Региональная диагностика |
| `account-closure-retention.md` | Decommission и закрытие аккаунта |
| `emergency-security-policy-disable.md` | Отключение политик безопасности |
| `cli-admin-tool.md` | Исходное требование к CLI-утилите |
| `support-sla.md` | Emergency Admin функции |
| `deployment-checklist.md` | P0 runbook и vedo-cli diagnose |
| `ticket-management-system.md` | Требования к жизненному циклу тикетов и CLI CRUD |
| `vision.md` | G5, F9, F12 |
| `project.yaml` | Состав стека |

---

## Принятые решения

- **Язык реализации:** Go (решение зафиксировано в `ADR-IMPL.STACK.vedo-cli-language-strategy`).
- **CLI-фреймворк:** Cobra (`spf13/cobra`) — решение зафиксировано в `ADR-IMPL.STACK.vedo-cli-framework-strategy`.
- **MFA-стратегия:** Keycloak как единый IdP: ROPG+TOTP для интерактивного режима, service account (`client_credentials`) для CI/CD — решение зафиксировано в `ADR-DES.SECURITY.cli-mfa-strategy`.
- **Модель распространения:** Single binary под каждую платформу, бинарные артефакты в релизе.
- **Хранение credentials backup storage:** HashiCorp Vault / AWS Secrets Manager / переменные окружения (fallback).

## Открытые вопросы

—
