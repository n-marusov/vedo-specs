# Блокирующие pre-prod проверки перед 100% rollout

| Атрибут | Значение |
|---------|----------|
| **ID** | REQ-FUN.PROCESS.preprod-release-gates |
| **Уровень** | FUN |
| **Атрибут качества** | Functionality |
| **Приоритет** | P0 |
| **Статус** | УТВЕРЖДЕНО |
| **Источник** | — |
| **Критерии приёмки** | — |

---

## Что описывает

Фиксирует обязательные pre-prod проверки, без прохождения которых запрещен переход rollout к 100% production-трафика.

## Область действия

- Все production релизы с изменением кода, конфигурации, схемы данных, security policy, API контрактов.
- Все major/minor релизы; hotfix допускает сокращенный набор по break-glass с обязательным post-factum review <= 24 часов.

## Блокирующие категории проверок

| Категория | Минимальный критерий pass | Блокирует rollout |
|---|---|---|
| Smoke | 100% pass по критичным health/auth/read/write smoke tests | Да |
| Security | 0 critical findings в SAST/SCA/secret scan, 0 failed policy checks | Да |
| Performance | p95/p99 не хуже baseline более чем на 20%/30% | Да |
| Data integrity | 100% pass миграционных и checksum проверок | Да |
| Backward compatibility | 100% pass обязательных contract tests | Да |

## Детальные пороги

### Smoke
- Все обязательные сценарии `auth`, `read`, `write`, `rollback-health` должны быть PASS.

### Security

| Проверка | Инструмент | Порог блокировки релиза |
|----------|------------|------------------------|
| SAST | Semgrep | 0 critical |
| SCA | `cargo-audit`, `govulncheck`, `safety`, `trivy` | 0 CRITICAL/HIGH |
| **Secret scan (CI)** | **gitleaks + GitLab Secret Detection** | **0 Block-секретов (см. REQ-SEC-80)** |
| **Secret scan (pre-commit)** | **gitleaks (рекомендован, не блокирует CI)** | **Предупреждение, commit не блокируется** |
| **SLSA provenance** | **slsa-verifier** | **Присутствует и верифицирован (Level 2)** |
| **Подпись артефакта** | **cosign** | **Верифицирована (cosign verify)** |
| **License compliance** | **cargo-deny, go-licenses, pip-licenses, license-checker** | **0 запрещённых лицензий (GPL, AGPL, LGPL, SSPL, CC-BY-NC) — см. REQ-SEC-100** |

#### SLSA / Подпись (детали)

Для production-релиза (тег `v*.*.*`):
1. Docker image должен быть подписан с помощью `cosign`.
2. Provenance (SLSA Level 2) должен быть приложен и верифицирован.
3. Верификация выполняется в CI перед деплоем.
4. Без подписи и верификации релиз блокируется.

Исключения:
- Non-production окружения (dev, staging) — подпись опциональна.
- On-premise сборки — подпись опциональна (рекомендована).

#### License Compliance (детали)

Перед релизом проверяются все зависимости (production и dev, если они поставляются с продуктом):

- **Block-лицензии** (GPL, AGPL, LGPL, SSPL, CC-BY-NC) → ❌ релиз блокируется.
- **Review-лицензии** (CDDL, EPL, JSON, WTFPL) → ⚠️ релиз блокируется до юридической проверки.
- Исключения оформляются через `licenses-exceptions.yaml` и требуют approval Security Lead.

**CI job пример:**
```yaml
license-scan:
  stage: security
  script:
    - cargo deny check licenses
    - go-licenses check ./...
    - pip-licenses --fail-on GPL,AGPL,SSPL --allow-only MIT,Apache-2.0,BSD
    - npx license-checker --failOn "GPL;AGPL;LGPL;SSPL"
  rules:
    - if: $CI_COMMIT_TAG
  allow_failure: false
```

#### Типы Block-секретов
- AWS/GCP/Yandex IAM keys
- GitHub tokens, GitLab tokens
- JWT signing keys (`VEDO_CORE_JWT_SECRET`)
- Production database passwords
- Stripe live keys, Slack webhooks
- Любой секрет, соответствующий `secret://` паттерну в коде

#### Исключения
- Test keys (помечены `test_*`, `example_*`) — Warning, не Block
- Плейсхолдеры (`<your-key>`, `YOUR_API_KEY`) — Ignore
- Файл `.gitleaks.toml` в корне репозитория (утверждён Security Lead)

### Performance
- `latency p95` <= baseline x1.2, `latency p99` <= baseline x1.3, `5xx` <= baseline x1.5.

### Data integrity
- Миграции должны быть reversible; checksum mismatch = hard fail.

### Compatibility
- Обязательные клиентские SDK/API сценарии должны пройти в полном объеме.

## Release gate правила

- Переход canary с 50% на 100% разрешен только при `preprod_gate_status=PASS`.
- Ручной override запрещен, кроме break-glass сценария P0 с двойным одобрением.
- Любой break-glass release должен иметь corrective MR и повторный полный pre-prod gate до следующего релиза.

## Артефакты доказательств

- `preprod-gate-report.json`
- pipeline/job URL
- commit SHA + container digest + helm chart version
- перечень failed/passed checks по категориям

Срок хранения evidence: >= 3 лет (WORM/append-only).

### Build Tools Version Gate

Для релизного тега (`v*.*.*`) проверяется соответствие версий build-инструментов pinned значениям:

| Инструмент | Файл конфигурации | Проверка |
|------------|-------------------|----------|
| Rust | `rust-toolchain.toml` | Должен совпадать с `RUST_VERSION` в CI |
| Go | `go.mod` | Должен совпадать с `GO_VERSION` в CI |
| Python | `.python-version` | Должен совпадать с `PYTHON_VERSION` в CI |
| Node.js | `.nvmrc` | Должен совпадать с `NODE_VERSION` в CI |

**Отклонение:** ❌ релиз блокируется (необходимо обновить файлы конфигурации).
**Grace period:** Не применяется к релизам (релиз должен использовать актуальную версию).

### UX Gate (для spec-driven разработки)

**Проверка** | **Условие прохождения**
--- | ---
Дизайн-система | `design/frontend.pen` — единый источник (Pencil-файл)
UX review для UI-изменений | Acceptance criteria утверждены UX Lead (метка `ux-approved` в PR)
Usability-тест (SUS/SEQ) | Для фич > 2 недель — результат теста приложен к MR (SUS ≥ 70)
Скриншот-тесты (Playwright) | Все тесты проходят (нет визуальных регрессий)
Accessibility (axe) | 0 critical/serious violations

**CI job пример:**
```yaml
ux-gate:
  stage: quality
  script:
    - npx playwright test --grep @visual
    - npx axe --threshold critical
    # design/frontend.pen — единый источник дизайн-токенов
  rules:
    - if: $CI_MERGE_REQUEST_ID
  allow_failure: false
```

**Исключения:** Experimental фичи (метка `experimental`) проходят только скриншот-тесты, без SUS/SEQ.

## Связанные артефакты

- `deployment-strategy.md`
- `deployment-checklist.md`
- `rollout-safety-gates.md`
- `authorization-regression-gates.md`
- `security-requirements.md` — REQ-SEC-55: Build Tools Version Policy, REQ-SEC-80: Secret Scanning, REQ-SEC-90: Verifiable Builds, REQ-SEC-100: License Compliance

## Бизнес-правила

- Production release без полного pre-prod gate считается невалидным.
- Если pre-prod gate провален, rollout должен быть остановлен и переведен в rollback path.
- Пороговые значения могут быть только ужесточены без отдельного ADR-обновления.
