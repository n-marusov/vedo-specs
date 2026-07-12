# SCA/SBOM Vulnerability Gating Specification — Technical Requirements v1.0

| Атрибут | Значение |
|---------|----------|
| **ID** | REQ-NFR.SECURITY.sca-sbom-gating |
| **Уровень** | NFR |
| **Атрибут качества** | Supportability |
| **Приоритет** | P0 |
| **Статус** | УТВЕРЖДЕНО |
| **Источник** | — |
| **Критерии приёмки** | — |

---

## Обзор

Документ определяет политику управления уязвимостями цепочки поставок (Supply Chain Security) для платформы VEDO Core. Регламентирует процедуры SCA-сканирования, генерации и хранения SBOM, правила блокировки merge/release на основе CVSS-порогов, процесс waiver (исключений) и мониторинг метрик безопасности цепочки поставок.

Политика применяется ко всем компонентам микросервисной архитектуры VEDO Core:

| Компонент | Язык/Экосистема | Инструмент SCA | Источник advisories |
|-----------|-----------------|----------------|---------------------|
| Frontend (Vue 3) | TypeScript / npm | `npm audit` | npm Advisory DB, GitHub Advisory DB, OSV |
| API Gateway | Go | `govulncheck` | Go VulnDB, NVD, OSV |
| Auth Service | Go | `govulncheck` | Go VulnDB, NVD, OSV |
| Ontology Service | Rust | `cargo-audit` | RustSec, NVD, OSV |
| Versioning Service | Rust | `cargo-audit` | RustSec, NVD, OSV |
| Metrics Service | Python | `safety` / `pip-audit` | PyPA Advisory DB, OSV |
| Docker base images | Multi-stage | `trivy image` | NVD, GitHub Advisory DB, OSV |
| Helm chart dependencies | YAML/Charts | `trivy config` | NVD, GitHub Advisory DB |

---

## 1. Политика SCA Gating

### 1.1 Пороги блокировки

| Severity | CVSS диапазон | Merge в main | Релиз | Примечание |
|----------|---------------|--------------|-------|------------|
| CRITICAL | 9.0–10.0 | Блокирует | Блокирует | Waiver невозможен |
| HIGH | 7.0–8.9 | Блокирует | Блокирует | Waiver макс. 7 дней |
| MEDIUM | 4.0–6.9 | Warning | Warning | Waiver при накоплении ≥5 |
| LOW | 0.1–3.9 | Info | Info | Не блокирует |

### 1.2 Lockfile drift

Обнаружение расхождения между декларированными зависимостями и lockfile (lockfile drift) блокирует **merge в main** и **релиз** для всех экосистем:

- Rust: `Cargo.lock` не соответствует `Cargo.toml` → `cargo-audit` error
- Go: `go.sum` не соответствует `go.mod` → `govulncheck` error
- Python: `requirements.txt` / `pyproject.toml` не соответствует lockfile → `safety` error
- Node.js: `package-lock.json` не соответствует `package.json` → `npm audit` error

### 1.3 Docker base images

Образы Docker с CRITICAL уязвимостями блокируют **релиз**. Проверка выполняется на этапе сборки через `trivy image`:

```yaml
# Пример команды проверки в CI
trivy image --severity CRITICAL --exit-code 1 \
  --ignore-unfixed \
  ${REGISTRY}/vedo-${SERVICE}:${CI_COMMIT_SHORT_SHA}
```

---

## 2. CI/CD Pipeline SCA Checks

### 2.1 Стадия security в GitLab CI

Пайплайн SCA-проверок интегрируется в существующую стадию `security` (определённую в `deploy/ci/gitlab-ci.yml` согласно ADR-DES.PROCESS.gitlab-ci-cd-strategy).

#### 2.1.1 Frontend (npm audit)

```yaml
sca-frontend:
  stage: security
  image: node:20-alpine
  script:
    - cd llm/src/frontend
    - npm ci
    - npm audit --audit-level=high
  artifacts:
    reports:
      dependency_scanning: gl-dependency-scanning-report.json
    paths:
      - llm/src/frontend/npm-audit.json
  rules:
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"
    - if: $CI_COMMIT_BRANCH == "main"
```

#### 2.1.2 Rust (cargo-audit)

```yaml
sca-rust:
  stage: security
  image: rust:1.77-slim-bookworm
  script:
    - cargo install cargo-audit
    - cd llm/src/ontology-service && cargo audit --deny warnings
    - cd llm/src/versioning-service && cargo audit --deny warnings
  artifacts:
    reports:
      dependency_scanning: gl-dependency-scanning-report.json
  rules:
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"
    - if: $CI_COMMIT_BRANCH == "main"
```

#### 2.1.3 Go (govulncheck)

```yaml
sca-go:
  stage: security
  image: golang:1.22-alpine
  script:
    - go install golang.org/x/vuln/cmd/govulncheck@latest
    - cd llm/src/api-gateway && govulncheck ./...
    - cd llm/src/auth-service && govulncheck ./...
  artifacts:
    reports:
      dependency_scanning: gl-dependency-scanning-report.json
  rules:
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"
    - if: $CI_COMMIT_BRANCH == "main"
```

#### 2.1.4 Python (safety)

```yaml
sca-python:
  stage: security
  image: python:3.12-slim
  script:
    - pip install safety
    - cd llm/src/metrics-service
    - safety check --full-report --threshold=702000  # HIGH severity
  artifacts:
    reports:
      dependency_scanning: gl-dependency-scanning-report.json
  rules:
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"
    - if: $CI_COMMIT_BRANCH == "main"
```

#### 2.1.5 Docker images (trivy)

```yaml
sca-docker:
  stage: security
  image: docker:24.0-cli
  services:
    - docker:24.0-dind
  script:
    - apk add --no-cache curl
    - curl -sfL https://raw.githubusercontent.com/aquasecurity/trivy/main/contrib/install.sh | sh -s -- -b /usr/local/bin v0.50.0
    - trivy image --severity CRITICAL,HIGH --exit-code 1 \
        --ignore-unfixed \
        ${REGISTRY}/vedo-frontend:${CI_COMMIT_SHORT_SHA}
    - trivy image --severity CRITICAL,HIGH --exit-code 1 \
        --ignore-unfixed \
        ${REGISTRY}/vedo-api-gateway:${CI_COMMIT_SHORT_SHA}
    - trivy image --severity CRITICAL,HIGH --exit-code 1 \
        --ignore-unfixed \
        ${REGISTRY}/vedo-ontology-service:${CI_COMMIT_SHORT_SHA}
    - trivy image --severity CRITICAL,HIGH --exit-code 1 \
        --ignore-unfixed \
        ${REGISTRY}/vedo-versioning-service:${CI_COMMIT_SHORT_SHA}
    - trivy image --severity CRITICAL,HIGH --exit-code 1 \
        --ignore-unfixed \
        ${REGISTRY}/vedo-auth-service:${CI_COMMIT_SHORT_SHA}
    - trivy image --severity CRITICAL,HIGH --exit-code 1 \
        --ignore-unfixed \
        ${REGISTRY}/vedo-metrics-service:${CI_COMMIT_SHORT_SHA}
  needs:
    - build-frontend
    - build-api-gateway
    - build-ontology-service
    - build-versioning-service
    - build-auth-service
    - build-metrics-service
  rules:
    - if: $CI_COMMIT_BRANCH == "main"
```

### 2.2 Интеграция с gates релиза

Согласно ADR-DES.PROCESS.deployment-integrity-strategy, каждый production deployment проходит автоматическую проверку целостности. SCA-проверки добавляются в evidence развёртывания:

- Отчёт SCA включается в состав deployment evidence (WORM/Object Lock storage)
- Любое расхождение между ожидаемым и фактическим digest артефакта с учётом SCA-результатов блокирует релиз
- Результаты SCA-сканирования подписываются и сохраняются как часть неизменяемого доказательства развёртывания

---

## 3. Процедура Waiver (Исключений)

### 3.1 Реестр waiver

Каждое исключение регистрируется в файле `human/artifacts/sca-waiver-registry.yaml`. Формат записи:

```yaml
waivers:
  - id: WVR-2026-001
    created_at: 2026-05-16T10:00:00Z
    expires_at: 2026-05-23T10:00:00Z
    status: active  # active | expired | revoked
    vulnerability:
      id: CVE-2026-12345
      severity: HIGH
      cvss: 7.5
      package: openssl-sys
      ecosystem: cargo
      component: ontology-service
      affected_version: ">=0.9.0, <0.9.15"
      fix_version: "0.9.15"
    justification: >
      Зависимость используется только в тестовом коде (feature-gated под cfg(test)).
      Потенциальный вектор атаки не достижим в production runtime.
    risk_assessment:
      exploitability: low          # low | medium | high | proven
      attack_vector: local         # network | adjacent | local | physical
      requires_auth: true
      production_exposure: false
    approved_by:
      role: security_lead
      name: <ФИО>
      date: 2026-05-16T10:00:00Z
    reviewed_by:
      role: tech_lead
      name: <ФИО>
      date: 2026-05-16T10:00:00Z
    auto_revoke: true
```

### 3.2 Правила waiver

| Параметр | CRITICAL | HIGH | MEDIUM |
|----------|----------|------|--------|
| Waiver возможен | Нет | Да | Да (при накоплении ≥5) |
| Макс. срок waiver | — | 7 дней | 30 дней |
| Требуется Security Lead | — | Да | Да |
| Требуется Tech Lead | — | Да | Да |
| Требуется DevOps | — | Да (оценка эксплуатации) | Нет |
| Auto-revoke по expiry | — | Да | Да |
| Накопление | — | Маск. 3 активных HIGH waiver | ≥5 → блокировка релиза |

### 3.3 Жизненный цикл waiver

1. **Обнаружение** — SCA-сканер фиксирует уязвимость в MR или CI
2. **Триаж** — Security Lead оценивает CVSS, exploitability, production exposure
3. **Обоснование** — Инициатор (Developer) описывает причину запроса waiver
4. **Согласование** — Security Lead + Tech Lead + DevOps утверждают или отклоняют
5. **Регистрация** — Запись в `sca-waiver-registry.yaml` в том же MR
6. **Мониторинг** — expiry отслеживается в CI: за 48 часов до истечения создаётся issue
7. **Продление** — Waiver продлевается через новый запрос с обновлённым обоснованием
8. **Revoke** — При появлении патча waiver автоматически отзывается, блокировка возобновляется

---

## 4. ФОРМАТ И ХРАНЕНИЕ SBOM

### 4.1 Выбор формата

VEDO Core использует **CycloneDX** как основной формат SBOM.

Обоснование выбора (согласно методологии ADR):
- CycloneDX является стандартом OWASP, нативно поддерживается инструментарием VEDO Core (Trivy, cargo-audit через cyclonedx-bom, npm audit через cyclonedx-npm)
- CycloneDX 1.5+ поддерживает компонентную разметку, типы компонентов (library, framework, application, container), граф зависимостей и метаданные сборки
- SPDX предпочтителен для лицензионной прозрачности; совместимость обеспечивается конвертацией через `spdx-to-cyclonedx` при необходимости

### 4.2 Генерация SBOM

SBOM генерируется на каждый компонент на этапе сборки (build stage):

| Компонент | Команда генерации | Выходной файл |
|-----------|-------------------|---------------|
| Frontend | `npx @cyclonedx/cyclonedx-npm --output-file sbom.cyclonedx.json` | `llm/src/frontend/sbom.cyclonedx.json` |
| Rust services | `cargo install cargo-cyclonedx && cd llm/src/ontology-service && cargo cyclonedx` | `llm/src/ontology-service/sbom.cyclonedx.json` |
| Go services | `go install github.com/CycloneDX/cyclonedx-gomod/cmd/cyclonedx-gomod@latest && cyclonedx-gomod mod -json -licenses` | `llm/src/api-gateway/sbom.cyclonedx.json` |
| Python | `pip install cyclonedx-bom && cyclonedx-py requirements -i requirements.txt -o sbom.cyclonedx.json` | `llm/src/metrics-service/sbom.cyclonedx.json` |
| Docker images | `trivy image --format cyclonedx --output sbom.cyclonedx.json ${IMAGE}` | `deploy/images/<service>/sbom.cyclonedx.json` |

### 4.3 Хранение SBOM

SBOM хранятся в GitLab Container Registry как multi-arch артефакты и дополнительно архивируются:

1. **Near-line** — артефакты CI/CD GitLab (retention 90 дней, соответствует ADR-DES.INFRA.telemetry-retention-strategy)
2. **Cold archive** — подписанные SBOM экспортируются в S3/MinIO Object Storage (retention: весь жизненный цикл продукта + 3 года)
3. **Deployment evidence** — SBOM прикрепляется к evidence развёртывания в WORM-хранилище (согласно ADR-DES.PROCESS.deployment-integrity-strategy)

### 4.4 Подпись SBOM

Каждый SBOM подписывается через `cosign` в CI:

```bash
cosign attest --type cyclonedx --predicate sbom.cyclonedx.json \
  ${REGISTRY}/vedo-${SERVICE}@${DIGEST}
```

Ключ подписи хранится в GitLab CI/CD masked variables. Верификация подписи выполняется перед релизом.

---

## 5. МЕТРИКИ И МОНИТОРИНГ

### 5.1 Ключевые показатели

| Метрика | Цель | Источник | Период замера |
|---------|------|----------|---------------|
| Mean Time To Remediate (MTTR) CRITICAL | ≤ 48 часов | GitLab CI + Jira/linear | Раз в сутки |
| Mean Time To Remediate (MTTR) HIGH | ≤ 7 дней | GitLab CI + Jira/linear | Еженедельно |
| Доля waiver от общего числа уязвимостей | ≤ 10% | Реестр waiver / Total CVEs | Раз в релиз |
| Возраст advisory DB | ≤ 24 часа с последнего обновления | Last updated timestamp | Раз в час (CI) |
| Процент компонентов с актуальным SBOM | 100% | SBOM registry scan | Раз в релиз |
| Coverable open vulnerabilities count | 0 CRITICAL, ≤ 3 HIGH | Текущее состояние | Непрерывно |
| Lockfile drift incidents | 0 | CI pipeline | На каждый MR |

### 5.2 Инструментарий мониторинга

- Метрики экспортируются в Prometheus через `sca_exporter`:
  - `vedo_sca_vulnerability_count{severity="critical", component="ontology-service"} 0`
  - `vedo_sca_mttr_hours{severity="high"} 72`
  - `vedo_sca_advisory_age_seconds 3600`
  - `vedo_sca_waiver_ratio 0.05`
- Дашборды в Grafana: панель `VEDO SCA Overview`
- Тревоги Prometheus Alertmanager:
  - `P0`: CRITICAL уязвимость не устранена > 48 часов
  - `P1`: HIGH уязвимость не устранена > 7 дней
  - `P2`: Возраст advisory DB > 48 часов

### 5.3 Доля waiver

Доля waiver рассчитывается как:
```
waiver_ratio = active_waivers / (active_waivers + resolved_vulnerabilities)
```
Порог срабатывания: `waiver_ratio > 0.10` генерирует предупреждение в CI.

При накоплении ≥5 MEDIUM уязвимостей без исправления и без waiver — генерируется блокирующий gate для релиза.

---

## 6. AIR-GAPPED DEPLOYMENT

### 6.1 Обновление advisory баз в offline-режиме

Согласно ADR-DES.INFRA.airgap-offline-deployment-strategy, VEDO Core поддерживает полностью изолированную среду без доступа к интернету. Для SCA-сканирования в air-gapped режиме предусмотрен механизм предварительной загрузки advisory баз.

```bash
# Шаг 1: На connected-машине подготовить bundle (выполняется вне контура)
vedo-cli airgap prepare --advisory-bundle ./airgap-bundle-$(date +%Y%m%d).tar.gz

# Шаг 2: Физически перенести bundle в изолированный контур

# Шаг 3: Установить bundle внутри контура
vedo-cli airgap import-advisories --bundle ./airgap-bundle-20260516.tar.gz
```

### 6.2 Состав advisory bundle

| Источник | Формат | Размер (приблиз.) | Частота обновления |
|----------|--------|-------------------|-------------------|
| RustSec (rustsec.org/advisories) | JSON/RustSec database | ~5 MB | Ежедневно |
| Go VulnDB (vuln.go.dev) | Go vulnerability database | ~3 MB | Ежедневно |
| PyPA Advisory DB (github.com/pypa/advisory-db) | YAML/JSON | ~10 MB | Ежедневно |
| npm Advisory DB (github.com/advisories) | GHSA format | ~8 MB | Ежедневно |
| NVD (nvd.nist.gov) | JSON 2.0 | ~200 MB (full) / ~2 MB (daily) | Ежедневно (delta) |
| GitHub Advisory DB (api.github.com/advisories) | GHSA format | ~15 MB | Ежедневно |
| Trivy vulnerability DB (ghcr.io/aquasecurity/trivy-db) | OCI image | ~500 MB | Ежедневно |

### 6.3 Флаги air-gapped режима

В air-gapped среде SCA-сканирование выполняется с флагами, исключающими online-проверки:

```yaml
# variables в GitLab CI для air-gapped runner
variables:
  VEDO_OFFLINE_MODE: "true"
  CARGO_AUDIT_OFFLINE: "--no-fetch"
  TRIVY_OFFLINE: "--cache-backend filesystem --db-repository /path/to/local/trivy-db"
  SAFETY_OFFLINE: "--key"
  GOVULNCHECK_OFFLINE: "-offline"
```

### 6.4 Валидация свежести advisory баз

Перед запуском SCA-сканирования выполняется проверка возраста последнего обновления:

```bash
vedo-cli airgap check-advisories --max-age-hours 48
exit_code=$?
if [ $exit_code -ne 0 ]; then
  echo "Advisory database is stale (> 48 hours). SCA scan blocked."
  exit $exit_code
fi
```

---

## 7. Роли и ответственность

### 7.1 Матрица RACI

| Действие | Security Lead | DevOps | Tech Lead | Developer |
|-----------|---------------|--------|-----------|-----------|
| Определение политики SCA | **R** | C | C | I |
| Настройка SCA-сканеров в CI | C | **R** | I | I |
| Триаж уязвимости CVE | **R** | C | C | I |
| Запрос waiver | A | I | C | **R** |
| Утверждение waiver CRITICAL | **R** | A | A | I |
| Утверждение waiver HIGH | **R** | C | **R** | I |
| Продление/отзыв waiver | **R** | I | C | I |
| Устранение уязвимости (fix) | I | C | C | **R** |
| Генерация SBOM | I | **R** | C | C |
| Подпись SBOM | **R** | **R** | I | I |
| Обновление advisory баз | I | **R** | I | I |
| Мониторинг метрик SCA | **R** | C | I | I |
| Аудит реестра waiver | **R** | C | I | I |
| Compliance отчётность | **R** | C | C | I |

*R — ответственный за выполнение, A — утверждающий, C — консультирующий, I — информируемый*

### 7.2 Описание ролей

| Роль | Обязанности | Требования | Назначение |
|------|-------------|------------|------------|
| **Security Lead** | Определение политики SCA, триаж уязвимостей, утверждение waiver, аудит реестра, compliance-отчётность | Сертификация OSCP/OSCE/ CISSP или 3+ лет опыта в AppSec | Назначается на уровне организации |
| **DevOps** | Настройка и поддержка SCA-сканеров в GitLab CI, обновление advisory баз в air-gapped среде, генерация SBOM, ротация ключей подписи | Опыт работы с GitLab CI, Docker, Trivy, CycloneDX | Назначается на уровне проекта |
| **Tech Lead** | Оценка технического воздействия уязвимости, согласование waiver, планирование исправлений, code review фиксов | Владеет архитектурой своего компонента | Назначается на каждый компонент/сервис |
| **Developer** | Устранение уязвимостей в своём коде, обновление зависимостей, подготовка обоснования для waiver | Член команды разработки | Владелец задачи |

---

## 8. Compliance Evidence

### 8.1 Артефакты для аудита

| Артефакт | Формат | Retention | Хранилище |
|----------|--------|-----------|-----------|
| SBOM CycloneDX каждого сборки | JSON | Жизненный цикл продукта + 3 года | WORM Object Storage |
| Отчёты SCA-сканирования | JSON / GitLab Dependency Scanning | 90 дней (near-line), 3 года (archive) | GitLab Artifacts + S3 |
| Подпись SBOM (cosign attestation) | DSSE/Rekor bundle | Жизненный цикл продукта + 3 года | WORM Object Storage |
| Реестр waiver | YAML | Постоянно (как часть репозитория) | Git LFS |
| Deployment evidence (включая SCA) | JSON + подпись | Жизненный цикл продукта + 3 года | WORM Object Storage |
| Audit log выпуска/отзыва waiver | JSON | 5 лет (согласно ADR-DES.INFRA.telemetry-retention-strategy) | Loki (audit tenant) |
| Метрики SCA (Prometheus) | Prometheus TSDB | 30 дней raw, 1 год aggregated | Prometheus / Thanos |
| Возраст advisory DB (timestamp) | Prometheus metric | 30 дней | Prometheus |

### 8.2 Compliance gates

```yaml
# gates-policy.yaml (дополнение существующей политики)
gates:
  - id: GATE-SCA-001
    name: No CRITICAL vulnerabilities
    check: sca_vulnerabilities{severity="critical"} == 0
    blocking: true
    applies_to: [release]

  - id: GATE-SCA-002
    name: No HIGH vulnerabilities without waiver
    check: sca_vulnerabilities{severity="high", waiver="none"} == 0
    blocking: true
    applies_to: [release]

  - id: GATE-SCA-003
    name: SBOM exists and signed
    check: sbom_exists_and_signed(component)
    blocking: true
    applies_to: [release]

  - id: GATE-SCA-004
    name: Advisory DB age < 48 hours
    check: sca_advisory_age_seconds < 172800
    blocking: true
    applies_to: [airgapped_release]

  - id: GATE-SCA-005
    name: No lockfile drift
    check: lockfile_drift(ecosystem) == 0
    blocking: true
    applies_to: [merge, release]

  - id: GATE-SCA-006
    name: MEDIUM waiver ratio < 10%
    check: sca_waiver_ratio < 0.10
    blocking: false  # warning
    applies_to: [release]
```

---

## 9. Исключения из политики

1. **Dev-окружения и локальная разработка** — SCA-сканирование не блокирует, но отчёт сохраняется
2. **Feature branch (не main)** — сканирование выполняется, блокировка только для CRITICAL
3. **Legacy dependencies с EOL более 12 мес.** — требуют отдельного approval Security Lead + Tech Lead
4. **Build-time-only зависимости** — не блокируют релиз, если доказано отсутствие propagation в runtime

---

## 10. License Compliance (дополнение к SCA)

Помимо SBOM и уязвимостей, SCA-пайплайн обязан проверять лицензионную совместимость зависимостей. Требования к лицензиям определены в `REQ-SEC-100` (`human/artifacts/requirements/REQ-NFR.SECURITY.security-requirements.md`).

### Правила

| Тип лицензии | Действие |
|--------------|----------|
| MIT, Apache-2.0, BSD, ISC, MPL-2.0, CC0, Unlicense | ✅ PASS |
| GPL, AGPL, LGPL, SSPL, CC-BY-NC | ❌ BLOCK |
| CDDL, EPL, JSON, WTFPL, OpenSSL | ⚠️ REVIEW (требует проверки юристом) |

### Инструменты

| Экосистема | Инструмент | Конфигурация |
|------------|------------|--------------|
| Rust | `cargo-deny` | `licenses.allow = ["MIT", "Apache-2.0", "BSD-2-Clause", "BSD-3-Clause", "ISC", "MPL-2.0"]` |
| Go | `go-licenses` | `go-licenses check --allowed_licenses MIT,Apache-2.0,BSD-3-Clause ./...` |
| Python | `pip-licenses` | `--allow-only "MIT;Apache 2.0;BSD;ISC" --fail-on "GPL;AGPL;LGPL;SSPL"` |
| Node.js | `license-checker` | `--failOn "GPL;AGPL;LGPL;SSPL"` |

### Исключения

Исключения из лицензионной политики хранятся в `licenses-exceptions.yaml` в корне репозитория. Изменения требуют approval Security Lead.

---

## 11. Open Questions

- Частота полного сканирования всех образов в Container Registry (периодический audit, не только при сборке) — требуется уточнение
- Интеграция с внешним SIEM для SCA-событий — deferred до запроса Enterprise-клиента
- Автоматическое создание CVE-тикетов через issue tracker — требуется выбор интеграции (GitLab Issues / Jira)
- Политика для transitive dependencies deep scan — требуется оценка производительности CI

---

## 12. Ссылки

- ADR-DES.PROCESS.gitlab-ci-cd-strategy — выбор GitLab CI как платформы CI/CD
- ADR-DES.PROCESS.deployment-integrity-strategy — проверка целостности развёртывания и evidence
- ADR-DES.INFRA.airgap-offline-deployment-strategy — поддержка изолированных сред
- ADR-DES.INFRA.telemetry-retention-strategy — политика хранения телеметрии
- `deploy/ci/gitlab-ci.yml` — текущая конфигурация GitLab CI
- `human/constraints/security.yaml` — глобальные ограничения безопасности VEDO Core
- `human/artifacts/requirements/REQ-NFR.SECURITY.security-requirements.md` — REQ-SEC-90: Verifiable Builds, REQ-SEC-100: License Compliance
- `human/artifacts/requirements/REQ-FUN.PROCESS.preprod-release-gates.md` — Security Gate с SLSA/provenance
- [CycloneDX Specification v1.5](https://cyclonedx.org/specification/overview/)
- [OWASP Dependency-Check](https://owasp.org/www-project-dependency-check/)
- [Trivy Documentation](https://trivy.dev/)
- [SLSA Framework](https://slsa.dev/)
- [Sigstore / cosign](https://sigstore.dev/)

---

## 13. Verifiable Builds & SLSA Compliance

### 13.1 Целевой уровень SLSA

| Уровень SLSA | Требование | Статус в VEDO Core |
|--------------|------------|---------------------|
| **Level 1** | Build script известен (Dockerfile, `.gitlab-ci.yml` в репозитории) | ✅ Выполнено |
| **Level 2** | Provenance подписан, build изолирован (GitLab CI, не локально) | ✅ **Цель для MVP** |
| **Level 3** | Reproducible builds (детерминированная сборка) | ⏸️ Отложено до Enterprise (по запросу) |
| **Level 4** | Два независимых билда, двухстороннее подтверждение | ❌ Не требуется |

### 13.2 Инструменты

| Инструмент | Назначение | Обязательность |
|------------|------------|----------------|
| **cosign** (Sigstore) | Подпись Docker image и provenance | ✅ Обязателен для production |
| **slsa-verifier** | Проверка подписи и provenance перед деплоем | ✅ Обязателен для production |
| **rekor** (прозрачный лог) | Хранение подписей (публичный экземпляр) | ⚠️ Рекомендован для Enterprise |
| **internal PKI** | Подпись в air-gapped среде | ⚠️ Опционально (для on-premise) |

### 13.3 Процесс сборки и подписи

1. **Сборка в изолированной среде:**
   - Все production-артефакты (Docker image, бинарные файлы) собираются **только в GitLab CI** (не локально).
   - CI job имеет OIDC-токен для аутентификации в Sigstore.

2. **Создание provenance (SLSA Level 2):**
   - Использовать `slsa-provenance` generator (или `cosign attest`).
   - Provenance должен содержать:
     - `sourceCommit` — хэш коммита из Git
     - `builderId` — идентификатор билдера (`https://gitlab.com/vedo-core/...`)
     - `dependencies` — список зависимостей (SBOM)
     - `invocation` — параметры сборки (Dockerfile, build args)

3. **Подпись артефакта (cosign):**
   ```bash
   cosign sign --key k8s://vedo-signing-secret $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA
   cosign attest --predicate provenance.json --key k8s://vedo-signing-secret $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA
   ```

4. **Хранение:**
   - Подпись и provenance хранятся в том же registry как отдельные артефакты.
   - Опционально: отправка в публичный Rekor (прозрачный лог Sigstore).

### 13.4 Верификация перед деплоем

Перед деплоем в production **обязательно** выполняется верификация:

```bash
# 1. Проверка подписи image
cosign verify --key vedo-pubkey.pem $CI_REGISTRY_IMAGE:$CI_COMMIT_TAG

# 2. Проверка provenance (SLSA Level 2)
slsa-verifier verify-image $CI_REGISTRY_IMAGE:$CI_COMMIT_TAG \
  --source-uri git@gitlab.com:vedo-core/$CI_PROJECT_NAME \
  --source-tag $CI_COMMIT_TAG
```

### 13.5 Пороги в CI (блокировка деплоя)

| Проверка | Условие блокировки | Действие |
|----------|---------------------|----------|
| Подпись image | Отсутствует для production-тега | ❌ Блокировка деплоя |
| Provenance (SLSA Level 2) | Отсутствует или невалиден | ❌ Блокировка деплоя |
| Source commit mismatch | Provenance ссылается на другой коммит | ❌ Блокировка деплоя |
| Builder ID mismatch | Provenance указывает не на GitLab CI | ❌ Блокировка деплоя |

### 13.6 Исключения

| Сценарий | Требование к подписи | Причина |
|----------|---------------------|---------|
| Non-production окружения (dev, staging) | Опционально (рекомендовано) | Упрощение отладки |
| On-premise (управляемый заказчиком) | Опционально | Заказчик доверяет своему registry |
| Air-gapped | Внутренняя PKI (не Sigstore) | Нет доступа к публичному Sigstore |
| Community / бесплатный тариф | Не требуется | Best-effort |

### 13.7 Reproducible builds (SLSA Level 3) — перспектива

Для достижения SLSA Level 3 необходимы дополнительные меры:

| Требование | Сложность | Статус |
|------------|-----------|--------|
| Фиксация версий всех build tools (Rust, Go, Python) | Средняя | ⏸️ Отложено |
| Детерминированная сборка Rust (`--remap-path-prefix`) | Высокая | ⏸️ Отложено |
| Исключение временных меток из бинарных артефактов | Низкая | ✅ Можно сделать |
| Проверка идентичности бинарных артефактов в CI | Высокая | ⏸️ Отложено |

**План введения SLSA Level 3:** По запросу Enterprise-клиента (не ранее 2027 года).

### 13.8 Соответствие существующим процессам

- Подпись SBOM (секция 4.4) использует тот же `cosign`, что и подпись артефактов.
- CI pipeline (секция 2) расширяется стадией `sign` и `verify` (см. ADR-DES.PROCESS.gitlab-ci-cd-strategy).
- Deployment evidence (секция 8) включает подпись и provenance как обязательные артефакты.
- Air-gapped (секция 6) использует внутреннюю PKI вместо публичного Sigstore.
