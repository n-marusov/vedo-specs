# Процедуры CI/CD

| Атрибут | Значение |
|---------|----------|
| **ID** | REQ-FUN.PROCESS.ci-cd-procedures |
| **Уровень** | FUN |
| **Атрибут качества** | Implementation |
| **Приоритет** | P0 |
| **Статус** | УТВЕРЖДЕНО |
| **Источник** | — |
| **Критерии приёмки** | — |

---

## Что описывает

Фиксирует ответ на вопрос T1.1: какие процедуры CI/CD обязательны для VEDO Core и какая CI/CD платформа используется.

VEDO Core поставляется с готовой конфигурацией GitLab CI в файле `.gitlab-ci.yml`. Все обязательные процедуры выполняются автоматически при push или merge request. Документация собирается с помощью Antora в многостраничный HTML site с поиском и навигацией.

## Платформа CI

Обязательная платформа по умолчанию: GitLab CI.

GitLab CI выбран как базовая поставка CI/CD, потому что он объединяет repository, merge requests, container registry, pipeline execution, artifacts, environments и manual approvals в одной платформе.

## Обязательные процедуры CI

| ID | Процедура | Триггер | Инструмент |
|----|-----------|---------|------------|
| CI-1 | Линтинг кода Rust, Go, Python, TypeScript | Push в любую ветку | `clippy`, `golangci-lint`, `ruff`, `eslint` |
| CI-2 | Модульные тесты | Push в любую ветку | `cargo test`, `go test`, `pytest`, `vitest` |
| CI-3 | Интеграционные тесты API | Merge Request в `main` | `pytest` + `requests` |
| CI-4 | Сборка Docker images | Push в `main` | `docker build` |
| CI-5 | Сборка документации | Push в `main` | `antora` |

## Обязательные процедуры CD

| ID | Процедура | Триггер | Среда |
|----|-----------|---------|-------|
| CD-1 | Deploy в dev | Автоматически после CI в `main` | dev |
| CD-2 | Smoke tests | После deploy в dev | dev |
| CD-3 | Deploy в staging | Вручную, кнопка | staging |
| CD-4 | Regression tests | После deploy в staging | staging |
| CD-5 | Deploy в production | Вручную, подтверждение | prod |
| CD-6 | Rollback | Вручную, до 5 версий | prod |

## Конфигурация GitLab CI

VEDO Core поставляется с `.gitlab-ci.yml`, включающим стадии:

```yaml
stages:
  - lint
  - test
  - build
  - docs
  - deploy-dev
  - deploy-staging
  - deploy-prod
```

Обязательные jobs линтинга:

- `rust-lint`: `cargo fmt --all -- --check`, `cargo clippy -- -D warnings`.
- `go-lint`: `golangci-lint run`.
- `python-lint`: `ruff check .`.
- `ts-lint`: `npm run lint`.

Обязательные test jobs:

- `rust-test`: `cargo test --all`.
- `go-test`: `go test -v ./...`.
- `python-test`: `pytest tests/unit/`.
- `ts-test`: `npm run test:unit`.
- `integration-test`: `pytest tests/integration/`, только merge requests.

Обязательные build jobs:

- `docker-build`: собирает и публикует `$CI_REGISTRY/vedo/vedo-core:$CI_COMMIT_SHORT_SHA`, только `main`.

Обязательный job документации:

- `docs`: устанавливает `@antora/cli` и `@antora/site-generator-default`, затем запускает `antora generate` для каждого playbook в `llm/src/docs/playbook-*.yml`.
- Documentation artifacts публикуются из `public/`.
- `docs-lint`: проверяет русскоязычный стиль, орфографию вне code blocks, ссылки и copy-paste готовность команд согласно `documentation-training.md`.

Обязательные deployment jobs:

- `deploy-dev`: автоматический Helm deploy в `vedo-dev` после CI на `main`.
- `smoke-test-dev`: health check после dev deploy.
- `deploy-staging`: ручной Helm deploy в `vedo-staging`.
- `regression-test`: ручные regression tests после staging deploy.
- `deploy-prod`: ручной Helm deploy в `vedo-prod`.
- `rollback-prod`: ручной Helm rollback в production.

Политика deployment strategy определена в `deployment-strategy.md`: rolling update используется по умолчанию, blue-green обязателен для production major updates при наличии ёмкости, а canary опционален для крупных инсталляций или A/B testing.

Политика environment configuration определена в `environment-configuration.md`: GitOps + Helm values files, при этом `k8s/dev/values.yaml`, `k8s/staging/values.yaml` и `k8s/prod/values.yaml` выбираются deploy jobs.

## Сборка документации Antora

VEDO Core использует Antora для многостраничной документации.

Обязательная структура repository:

```text
llm/src/docs/
├── playbook-admin.yml      # Antora playbook: admin-guide
├── playbook-dev.yml         # Antora playbook: developer-guide
├── playbook-integrator.yml  # Antora playbook: integrator-guide
├── playbook-user.yml        # Antora playbook: user-guide
├── Dockerfile               # Multi-playbook Docker build (ARG PLAYBOOK)
├── docker-compose.docs.yml   # 4 services, one per playbook
├── scripts/
│   ├── build-docs.mjs       # Build script (PLAYBOOK env var or all)
│   ├── check-links.mjs
│   ├── check-style.mjs
│   └── check-snippets.mjs
└── tests/
    └── docs.test.mjs

docs/antora/
├── admin-guide/
│   ├── antora.yml
│   └── modules/ROOT/{pages,nav.adoc}
├── developer-guide/
│   ├── antora.yml
│   └── modules/ROOT/{pages,nav.adoc}
├── integrator-guide/
│   ├── antora.yml
│   └── modules/ROOT/{pages,nav.adoc}
├── user-guide/
│   ├── antora.yml
│   └── modules/ROOT/{pages,nav.adoc}
├── supplemental-ui/
└── antora-ui-loader-3.1.14.tgz
│   └── api/
│       ├── pages/
│       └── nav.adoc
└── suppl/
    └── images/
```

Обязательные правила Antora:

- Все страницы пишутся в AsciiDoc (`.adoc`).
- Навигация задаётся `nav.adoc` в каждом module.
- Images хранятся в `docs/suppl/images/`.
- Documentation обновляется при каждом push в `main` через CI stage `docs`.
- Сгенерированный HTML site размещается в `public/`.

## Формулировка для заказчика

VEDO Core поставляется с готовой конфигурацией GitLab CI, включающей линтинг, тестирование, сборку, deploy и rollback.

Документация, включая user guide и developer guide, автоматически собирается через Antora и публикуется как многостраничный HTML site. Для работы с документацией не требуется дополнительных инструментов у конечного пользователя.

Заказчик может использовать `.gitlab-ci.yml` как есть или адаптировать под свои окружения, например добавить нагрузочное тестирование или изменить namespaces dev/staging/prod.

## Бизнес-правила

- GitLab CI является платформой CI/CD по умолчанию для поставки VEDO Core.
- `.gitlab-ci.yml` должен быть включён в product repository.
- Push в любую ветку должен запускать линтинг и unit tests.
- Merge requests в `main` должны запускать API integration tests.
- Push в `main` должен собирать Docker images.
- Push в `main` должен собирать документацию Antora.
- Deploy в dev выполняется автоматически после успешного CI на `main`.
- Deploy в staging выполняется вручную.
- Deploy в production выполняется вручную и требует подтверждения.
- Production rollback выполняется вручную и поддерживает rollback до 5 версий.
- Documentation source должен использовать AsciiDoc и Antora modules.
- Antora generated site должен публиковаться из artifacts `public/`.
- Documentation CI должен включать Antora build, link checker, spell/style lint и безопасную проверку командных snippets.
- Изменения функциональности, API, CLI, deployment или user workflow должны обновлять документацию в том же MR/PR.

## Deployment evidence

Deployment evidence фиксируется автоматически CI/CD пайплайном, без возможности ручного пропуска или post-factum формирования.

### Состав evidence

| Поле | Описание | Источник |
|------|----------|----------|
| **Deployment ID** | Уникальный ID развёртывания | CI/CD пайплайн |
| **Timestamp** | Время начала и завершения | CI/CD пайплайн |
| **Operator** | Кто инициировал (человек/триггер) | `GITLAB_USER_LOGIN` |
| **Artifact reference** | Commit SHA, тег образа, версия Helm chart | CI/CD переменные |
| **Target environment** | Имя окружения, имена узлов/подов | Helm release, Kubernetes API |
| **Deployment result** | Успех/неудача, количество обновлённых узлов | CI/CD + Kubernetes API |
| **Verification status** | Результаты post-deploy smoke tests | CI/CD пайплайн |
| **Rollback reference** | Ссылка на evidence предыдущего deployment | CI/CD пайплайн |

### Хранение

- Immutable storage (S3 с Object Lock / WORM), отдельно от production.
- Каждая запись подписана SHA-256; ручное изменение/удаление невозможно.
- Append-only; evidence доступно для аудита без модификации.
- Rollback-развёртывания фиксируются с теми же требованиями.

### Обоснование

Урок Knight Capital (2013): $440M потеряно за 45 минут из-за того, что 1 из 8 серверов не получил обновление, а deployment-процесс не обеспечил проверку всех узлов. Автоматическая фиксация evidence на каждом узле предотвращает этот класс инцидентов.

### Что не допускается

- Ручное заполнение evidence post-factum.
- Флаг `--skip-evidence`.
- Хранение evidence в том же хранилище, что и production, без WORM.
- Отсутствие evidence для rollback-развёртываний.

## SLSA и supply chain security

### Требуемый уровень

| Параметр | MVP | Release 1.0 (цель) |
|----------|-----|-------------------|
| Минимальный уровень SLSA | SLSA Level 2 | SLSA Level 3 |
| Подпись provenance | Обязательно (`intoto.json`) | Обязательно |
| Builder identity | `gitlab-runner@vedo-core` | `gitlab-runner@vedo-core` |

### Блокируемые условия

Сборка блокируется, если:
- Нет provenance в формате `intoto.json` для каждого Docker image.
- Source commit не верифицирован.
- Builder identity не соответствует `gitlab-runner@vedo-core`.

### Provenance включает

- Source commit hash
- Builder ID
- Список build dependencies (SBOM)
- Timestamp

### Хранение SBOM и provenance

| Артефакт | Срок хранения | Хранилище |
|----------|---------------|-----------|
| SBOM | 3 года | WORM-бакет (Object Lock), отдельные credentials от production |
| Provenance (SLSA) | 3 года | WORM-бакет |
| Container image | 1 год (активная поддержка) | Container Registry |
| Helm chart | Бессрочно для LTS-версий | Git + Chart Museum |

### Trusted builders

Допускаются: GitLab CI runners (self-hosted, изолированные), GitHub Actions (только self-hosted runner для production). Запрещены: public runners без изоляции, любые непроверяемые билд-серверы.

## Developer onboarding (DX gate)

CI/CD пайплайн включает make targets для сокращения времени до первого коммита нового разработчика:

| Сценарий | Целевое время | make target |
|----------|---------------|-------------|
| Поднять локальный стек (docker-compose) | 15 минут | `make dev-setup` |
| Прогнать все тесты | 10 минут | `make test` |
| Воспроизвести RDF/import (100K триплетов) | 5 минут | `make demo-ontology` |
| Полный onboarding до первого коммита | 90 минут | README + Antora Developer Guide |

`make dev-setup` автоматически проверяет наличие зависимостей (Docker, Rust, Go, Python, Node.js). `make demo-ontology` загружает тестовую онтологию (Pizza ontology).

## Open questions

- Нет открытых вопросов по T1.1.
