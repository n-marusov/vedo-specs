# Архитектурные решения (ADR) — VEDO Core

В этой директории хранятся записи архитектурных решений (Architecture Decision Records) проекта VEDO Core.

## Правила именования файлов

Файлы именуются по шаблону `<ADR-ID>.md`, где `ADR-ID` — уникальный идентификатор решения.

**Формат идентификатора:**
```
ADR-<LEVEL>.<AREA>.<semantic-tag>
```

Где:
- `LEVEL` = `BIZ` | `DES` | `IMPL`
- `AREA` = `API` | `DATA` | `INFRA` | `SECURITY` | `UI` | `PROCESS` | `INTEGRATION` | `STACK` | `OPS` | `DOC` | `SUP`
- `semantic-tag` — короткая англоязычная метка в kebab-case (2–5 слов), отражающая суть выбора

## Структура ADR

Каждый ADR должен содержать следующие обязательные поля:

- **Статус:** `[ЧЕРНОВИК | ПРЕДЛОЖЕНО | ПРИНЯТО | УСТАРЕЛО | ЗАМЕНЕНО]`
- **Дата:** `ГГГГ-ММ-ДД`
- **Контекст:** описание контекста проблемы
- **Требование-источник:** ссылки на файлы или ID требований
- **Решение:** краткое описание выбора (должно соответствовать semantic-tag)
- **Рассмотренные альтернативы:** перечисление рассмотренных вариантов (если применимо)
- **Последствия:** положительные и отрицательные последствия + способы смягчения

## Паттерны semantic-tag

| Паттерн | Описание | Пример |
|---------|----------|--------|
| `-vs-` / `-vs-...-vs-` | Выбор между альтернативами | `postgres-vs-mysql` |
| `-or-` | Равнозначные варианты | `redis-or-memcached` |
| `-tradeoff` | Компромисс между качествами | `latency-vs-consistency-tradeoff` |
| `-adoption` | Внедрение технологии без альтернатив | `kubernetes-adoption` |
| `-mandate` | Вынужденное решение | `gdpr-mandate` |
| `-strategy` / `-approach` / `-pattern` | Выбор подхода | `cache-strategy`, `saga-pattern` |
| `-evolution` / `-migration` | Изменение существующего | `microservices-evolution` |
| `-scope` / `-boundary` | Определение границ | `context-boundary` |

## Правила

1. **Уникальность ID** — каждый ADR-ID должен быть уникальным в рамках проекта
2. **LEVEL соответствует типу решения:** `BIZ` — бизнес-решения, `DES` — проектные, `IMPL` — реализационные
3. **Максимальная длина semantic-tag** — 40 символов
4. **Запрещены пробелы** в ID, только дефисы и точки
5. **Перед созданием нового ADR** проверьте существующие на пересечение темы
6. **Изменения принятых ADR** оформляются через обновление статуса (заменено/устарело) и создание нового ADR

## Список ADR

| ID | Название | Статус | Дата |
|----|----------|--------|------|
| `ADR-DES.API.swagger-ui-dev-only-strategy` | Swagger UI dev-only | Принято | 2026-07-18 |
| `ADR-DES.INFRA.monolith-vs-microservices` | Монолит vs Микросервисы | Принято | 2026-05-08 |
| `ADR-IMPL.STACK.frontend-vue-strategy` | Фронтенд на Vue 3 | Принято | 2026-05-09 |
| `ADR-IMPL.PROCESS.ui-design-pencil-adoption` | Pencil.dev для дизайна | Принято | 2026-05-09 |
| `ADR-IMPL.STACK.antora-docs-adoption` | Antora для документации | Принято | 2026-05-11 |
| `ADR-DES.PROCESS.gitlab-ci-cd-strategy` | GitLab CI/CD | Принято | 2026-05-11 |
| `ADR-DES.PROCESS.deployment-strategy-policy` | Стратегии развёртывания | Принято | 2026-05-11 |
| `ADR-DES.PROCESS.gitops-helm-config-strategy` | GitOps + Helm | Принято | 2026-05-11 |
| `ADR-IMPL.STACK.ontology-rust-strategy` | Rust для Ontology Service | Принято | 2026-05-09 |
| `ADR-IMPL.STACK.version-control-rust-strategy` | Rust для Versioning Service | Принято | 2026-05-09 |
| `ADR-IMPL.STACK.auth-service-go-strategy` | Go для Auth Service | Принято | 2026-05-09 |
| `ADR-IMPL.STACK.metrics-python-strategy` | Python для Metrics Service | Принято | 2026-05-09 |
| `ADR-IMPL.STACK.microservice-language-stack-strategy` | Языковой стек микросервисов | Принято | 2026-05-19 |
| `ADR-IMPL.STACK.port-mapping-strategy` | Схема портов | Предложено | 2026-05-19 |
| `ADR-IMPL.PROCESS.repository-layout-strategy` | Структура репозитория | Принято | 2026-05-19 |
| `ADR-IMPL.PROCESS.c4-notation-adoption` | C4-нотация | Принято | 2026-05-09 |
| `ADR-DES.INFRA.otel-observability-strategy` | OpenTelemetry стек | Принято | 2026-05-09 |
| `ADR-DES.INFRA.vedo-cli-diagnostics-entrypoint` | CLI диагностика | Принято | 2026-05-12 |
| `ADR-DES.INFRA.vedo-cli-admin-boundary` | CLI администрирование | Принято | 2026-05-12 |
| `ADR-DES.INFRA.critical-alerts-strategy` | Критичные алерты | Принято | 2026-05-10 |
| `ADR-DES.INFRA.telemetry-retention-strategy` | Хранение телеметрии | Принято | 2026-05-10 |
| `ADR-DES.DATA.account-closure-retention-strategy` | Хранение при закрытии аккаунта | Принято | 2026-05-10 |
| `ADR-DES.DATA.decommission-export-format-strategy` | Формат экспорта при деактивации | Принято | 2026-05-10 |
| `ADR-DES.DATA.secure-erase-responsibility-strategy` | Безопасное удаление | Принято | 2026-05-10 |
| `ADR-DES.INFRA.data-residency-region-strategy` | Резидентность данных | Принято | 2026-05-10 |
| `ADR-DES.INFRA.airgap-offline-deployment-strategy` | Air-gapped развёртывание | Принято | 2026-05-10 |
| `ADR-DES.INFRA.sustainability-energy-efficiency-strategy` | Энергоэффективность | Принято | 2026-05-10 |
| `ADR-DES.API.protocol-stack-strategy` | Стек протоколов API | Принято | 2026-05-09 |
| `ADR-DES.API.graphql-sparql-split-strategy` | GraphQL + SPARQL | Принято | 2026-05-12 |
| `ADR-DES.API.sparql-query-language-strategy` | SPARQL как язык запросов | Принято | 2026-05-12 |
| `ADR-DES.API.backward-compatibility-strategy` | Обратная совместимость API | Принято | 2026-05-10 |
| `ADR-DES.DATA.storage-stack-strategy` | Стек хранения данных | Принято | 2026-05-09 |
| `ADR-IMPL.INFRA.docker-adoption` | Docker | Принято | 2026-05-09 |
| `ADR-BIZ.INFRA.saas-deployment-strategy` | SaaS стратегия | Принято | 2025-03-15 |
| `ADR-DES.INFRA.recovery-objectives-mandate` | RTO/RPO мандат | Принято | 2025-03-15 |
| `ADR-DES.INFRA.backup-policy-strategy` | Политика бэкапов | Принято | 2026-05-10 |
| `ADR-DES.PROCESS.major-version-migration-strategy` | Миграция мажорных версий | Принято | 2026-05-10 |
| `ADR-DES.PROCESS.rollback-data-strategy` | Откат данных | Принято | 2026-05-10 |
| `ADR-DES.INFRA.fault-tolerance-strategy` | Отказоустойчивость | Принято | 2025-03-15 |
| `ADR-DES.UI.wcag-accessibility-strategy` | WCAG доступность | Принято | 2026-05-17 |
| `ADR-DES.UI.design-system-customization-strategy` | Кастомизация дизайн-системы | Принято | 2026-05-18 |
| `ADR-DES.UI.data-loss-prevention-strategy` | Защита от потери данных | Принято | 2026-05-10 |
| `ADR-DES.UI.dangerous-actions-recovery-strategy` | Восстановление опасных действий | Принято | 2026-05-10 |
| `ADR-DES.UI.version-context-visibility-strategy` | Видимость контекста версии | Принято | 2026-05-10 |
| `ADR-DES.UI.public-landing-architecture` | Архитектура публичного лендинга | Предложено | 2026-08-01 |
| `ADR-DES.UI.import-export-safety-strategy` | Безопасность импорта/экспорта | Принято | 2026-05-10 |
| `ADR-DES.UI.error-feedback-strategy` | Обратная связь об ошибках | Принято | 2026-05-10 |
| `ADR-DES.UI.navigation-state-strategy` | Состояние навигации | Принято | 2026-05-10 |
| `ADR-DES.UI.ontology-mental-model-strategy` | Ментальная модель онтологии | Принято | 2026-05-10 |
| `ADR-DES.UI.usability-release-gates-strategy` | Юзабилити-гейты релиза | Принято | 2026-05-10 |
| `ADR-DES.DOC.localization-language-policy` | Языковая политика | Принято | 2026-05-12 |
| `ADR-DES.OPS.support-sla-and-escalation-strategy` | SLA и эскалация | Принято | 2026-05-12 |
| `ADR-DES.INFRA.ontology-publishing` | Публикация онтологий | Принято | 2026-05-13 |
| `ADR-DES.SECURITY.public-ontology-access` | Публичный доступ к онтологиям | Принято | 2026-05-13 |
| `ADR-DES.SECURITY.gitlab-like-organization-model` | GitLab-like модель организации | Принято | 2026-05-13 |
| `ADR-DES.API.unified-root-endpoint-adoption` | Единый корневой endpoint | Принято | 2026-05-13 |
| `ADR-DES.PROCESS.decommission-drill-strategy` | Учения по деактивации | Принято | 2026-05-13 |
| `ADR-DES.PROCESS.deployment-integrity-strategy` | Целостность развёртывания | Принято | 2026-05-13 |
| `ADR-DES.INFRA.restore-drill-strategy` | Учения по восстановлению | Принято | 2026-05-13 |
| `ADR-DES.UI.accessibility-manual-testing-strategy` | Ручное тестирование доступности | Принято | 2026-05-13 |
| `ADR-DES.SECURITY.destructive-command-guardrails` | Guardrails для разрушительных команд | Принято | 2026-05-13 |
| `ADR-DES.SECURITY.mfa-critical-ops-mandate` | MFA для критических операций | Принято | 2026-05-13 |
| `ADR-DES.INFRA.emergency-kill-switch-strategy` | Аварийный выключатель | Принято | 2026-05-13 |
| `ADR-DES.INFRA.immutable-backup-strategy` | Неизменяемые бэкапы | Принято | 2026-05-13 |
| `ADR-DES.SECURITY.break-glass-access-strategy` | Break-glass доступ | Принято | 2026-05-13 |
| `ADR-BIZ.PROCESS.vendor-retirement-notice-mandate` | Уведомление о завершении услуг | Принято | 2026-05-13 |
| `ADR-DES.INTEGRATION.saga-pattern-strategy` | Saga паттерн | Принято | 2026-05-13 |
| `ADR-DES.PROCESS.merge-request-strategy` | Merge Request процесс | Принято | 2026-05-16 |
| `ADR-DES.SECURITY.supply-chain-vulnerability-policy` | Политика уязвимостей цепочки поставок | Принято | 2026-05-16 |
| `ADR-DES.INFRA.edge-region-failover-strategy` | Edge/региональный failover | Принято | 2026-05-16 |
| `ADR-DES.INFRA.support-metadata-isolation-strategy` | Изоляция метаданных поддержки | Принято | 2026-05-16 |
| `ADR-DES.DATA.config-overrides-retention-strategy` | Хранение конфигураций | Предложено | 2026-05-23 |
| `ADR-IMPL.DATA.support-db-sync-strategy` | Синхронизация Support DB | Предложено | 2026-05-23 |
| `ADR-IMPL.SECURITY.bola-bfla-negative-tests-mandate` | BOLA/BFLA негативные тесты | Принято | 2026-05-16 |
| `ADR-IMPL.PROCESS.deployment-checklist-mandate` | Чеклист развёртывания | Принято | 2026-05-17 |
| `ADR-IMPL.SECURITY.parser-query-fuzz-gates-mandate` | Фаззинг-гейты парсеров | Принято | 2026-05-17 |
| `ADR-DES.PROCESS.staged-rollout-auto-rollback-strategy` | Staged rollout + auto-rollback | Принято | 2026-05-17 |
| `ADR-DES.INFRA.control-plane-isolation-strategy` | Изоляция control plane | Принято | 2026-05-17 |
| `ADR-DES.UI.incident-communication-sla-strategy` | SLA коммуникации инцидентов | Принято | 2026-05-17 |
| `ADR-DES.UI.degraded-mode-ux-strategy` | UX режима деградации | Принято | 2026-05-17 |
| `ADR-DES.SECURITY.privileged-access-jit-pam-strategy` | JIT/PAM доступ | Принято | 2026-05-17 |
| `ADR-DES.SECURITY.incident-secret-rotation-strategy` | Ротация секретов при инцидентах | Принято | 2026-05-17 |
| `ADR-DES.SECURITY.authorization-policy-gates-strategy` | Гейты авторизации | Принято | 2026-05-17 |
| `ADR-DES.API.write-idempotency-strategy` | Идемпотентность записи | Принято | 2026-05-17 |
| `ADR-DES.PROCESS.incident-response-slo-strategy` | SLO реагирования на инциденты | Принято | 2026-05-17 |
| `ADR-DES.PROCESS.preprod-release-gates-strategy` | Pre-prod гейты | Принято | 2026-05-17 |
| `ADR-DES.SECURITY.emergency-policy-disable-strategy` | Аварийное отключение политик | Принято | 2026-05-17 |
| `ADR-BIZ.PROCESS.decommission-notification-mandate` | Уведомление о деактивации | Принято | 2026-05-17 |
| `ADR-DES.DATA.decommission-archive-retention-strategy` | Архив деактивации | Принято | 2026-05-17 |
| `ADR-IMPL.STACK.vedo-cli-language-strategy` | Язык для vedo-cli | Принято | 2026-05-18 |
| `ADR-IMPL.SECURITY.vedo-cli-credentials-strategy` | Credentials для vedo-cli | Принято | 2026-05-18 |
| `ADR-IMPL.INTEGRATION.commenting-service-architecture` | Архитектура сервиса комментариев | Принято | 2026-05-18 |
| `ADR-IMPL.OPS.ticket-management-system-architecture` | Архитектура системы тикетов | Принято | 2026-05-18 |
| `ADR-IMPL.STACK.ticketing-language-strategy` | Язык для системы тикетов | Принято | 2026-05-18 |
| `ADR-IMPL.STACK.vedo-cli-framework-strategy` | Фреймворк для vedo-cli | Принято | 2026-05-19 |
| `ADR-DES.SECURITY.cli-mfa-strategy` | MFA в CLI | Принято | 2026-05-19 |
| `ADR-IMPL.UI.volt-vs-primevue-vs-vuetify` | Выбор UI библиотеки | Принято | 2026-05-22 |
| `ADR-IMPL.SUP.ticket-operations-channels` | Каналы операций с тикетами | Принято | 2026-05-24 |
| `ADR-DES.UI.ux-delight-implementation-strategy` | UX Delight реализация | Принято | 2026-05-23 |
| `ADR-DES.API.cypher-query-language-adoption` | CYPHER язык запросов | Принято | 2026-05-24 |
| `ADR-DES.INTEGRATION.mcp-server-query-adoption` | MCP-сервер | Принято | 2026-05-24 |
| `ADR-DES.SECURITY.nl-query-opt-in-mandate` | Opt-in для NL запросов | Принято | 2026-05-24 |
| `ADR-DES.INFRA.ai-orchestration-service-strategy` | Выделение AI-оркестрации из Gateway | Предложено | 2026-07-16 |
| `ADR-DES.API.organization-rest-endpoints` | Канонический REST-контракт organization model | Принято | 2026-07-21 |
| `ADR-DES.DATA.uuid-identifiers-for-groups-projects-mandate` | UUID идентификаторы для групп и проектов | Принято | 2026-07-23 |
