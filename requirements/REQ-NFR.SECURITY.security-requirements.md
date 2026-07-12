# Требования безопасности

| Атрибут | Значение |
|---------|----------|
| **ID** | REQ-NFR.SECURITY.security-requirements |
| **Уровень** | NFR |
| **Атрибут качества** | Functionality |
| **Приоритет** | P0 |
| **Статус** | УТВЕРЖДЕНО |
| **Источник** | — |
| **Критерии приёмки** | — |

---

## Emergency Admin (Break-Glass Access)

### Управление emergency-доступом

Emergency Admin пароль разделяется через Shamir Secret Sharing M-of-N (минимум 3 из 5 частей для восстановления). Каждая часть хранится у отдельного доверенного лица (security-команда).

### Хранение частей Shamir-разделённого пароля

- Части пароля хранятся вне VEDO Core: каждая часть — в индивидуальном защищённом хранилище доверенного лица (HashiCorp Vault, password manager с MFA, hardware token или сейф).
- VEDO Core не хранит полный пароль Emergency Admin в какой-либо базе данных или конфигурации.
- Для восстановления пароля требуется M-of-N объединение частей в безопасной среде (offline или trusted build agent).
- Пароль ротируется каждые 90 дней и после каждого использования.
- Сброс пароля требует повторного Shamir-разделения и рассылки новых частей держателям.

### Endpoint /emergency/login

- Endpoint **не зависит от Keycloak** и доступен при полной недоступности IdP.
- Аутентификация: полный пароль, восстановленный из M-of-N частей Shamir.
- Сессия: строго ограничена по времени (max 30 минут), автоматическое завершение (logout) по истечении.
- После успешной аутентификации доступны только: `emergency readonly`, `restore`, `diagnose`, управление admin-учётками.

### Аудит и уведомления

- Все операции Emergency Admin логируются с actor, timestamp, использованными операциями, результатом, correlation ID.
- Выход из emergency-сессии автоматически завершает сессию; повторный вход требует повторного ввода полного пароля.
- Немедленное уведомление security-команды при каждой активации Emergency Admin.
- Любое использование L1 (Emergency Admin) или escalation на L2/L3 фиксируется в immutable audit log.

---

## REQ-SEC-50: SCA Toolchain Version Pinning

**Описание:** Все инструменты Software Composition Analysis должны использовать фиксированные версии, определённые в ADR-DES.SECURITY.supply-chain-vulnerability-policy.

**Обоснование:** Обеспечение воспроизводимости проверок безопасности и предотвращение breaking changes в CI.

**Pinned версии:**

| Инструмент | Версия |
|------------|--------|
| `cargo-audit` | 0.21.2 |
| `govulncheck` | 1.0.4 |
| `safety` | 3.4.0 |
| `trivy` | 0.69.6 |

**Критерии проверки:**
- `cargo-audit` запускается с версией, указанной в CI (проверка по `--version`).
- `govulncheck` — проверка версии в `go install` команде.
- `safety` — установка через `pip install safety==X.Y.Z`.
- `trivy` — использование Docker-образа с тегом конкретной версии.

**Ответственный:** DevOps / Security Lead.

**Частота проверки:** CI при каждом запуске (автоматически) + аудит раз в квартал.

---

## REQ-SEC-55: Build Tools Version Policy

**Описание:** Все build-инструменты (Rust, Go, Python, Node.js) должны использовать фиксированные версии, обновляемые по политике с grace period.

**Требования:**

| Инструмент | Pinned версия | Grace period | Обновление | Ответственный |
|------------|---------------|--------------|------------|---------------|
| Rust | 1.85.x | 90 дней | Ежеквартально (Renovate) | Tech Lead |
| Go | 1.23.x | 90 дней | Ежеквартально (Renovate) | Tech Lead |
| Python | 3.12.x | 90 дней | Ежеквартально (Renovate) | Tech Lead |
| Node.js | 20.x LTS | 90 дней | Ежеквартально (Renovate) | Tech Lead |

**Исключения:**
- Security patches — обновление в течение 7 дней (автоматический PR).
- Breaking changes — требуют отдельного approval от Tech Lead.

**Измерение соблюдения:**
- CI проверяет версии инструментов в начале каждого пайплайна.
- Отклонение от pinned версии → warning, не блокирует сборку (чтобы не ломать legacy ветки).
- Для тегов (`v*.*.*`) CI проверяет строгое соответствие (иначе релиз блокируется).

**Хранение версий:**
- `.rust-toolchain.toml`, `go.mod`, `.python-version`, `.nvmrc` в корне репозитория.
- `.gitlab-ci.yml` использует эти же версии.

---

## REQ-SEC-70: DDoS Protection

**Описание:** VEDO Core должен быть защищён от распределённых атак на уровнях L3/L4 и L7.

**Требования:**

| Уровень | Тип атаки | Мера защиты | Порог |
|---------|-----------|-------------|-------|
| L3/L4 | SYN flood, UDP amplification, ICMP flood | Cloud provider DDoS mitigation | Включено по умолчанию |
| L7 | HTTP flood (volumetric) | Rate limiting (per IP, per tenant) | Неавторизованные: 100 req/min<br>Авторизованные: 1000 req/min |
| L7 | Slowloris (slow headers/body) | `client_header_timeout: 5s`, `client_body_timeout: 10s` | — |
| L7 | WebSocket Slowloris | `proxy_read_timeout: 60s`, handshake timeout: 10s | — |
| L7 | HTTP/2 Rapid Reset | Ограничение на количество RST_STREAM frames | 100 frames/sec per connection |
| L7 | SPARQL / GraphQL complex queries | Query complexity limit + rate limiting | 30 req/min per IP |

**Обязанность:** Защита L3/L4 — cloud-провайдер. WAF и rate limiting — VEDO Core.

**Тестирование:** Chaos-тесты с симуляцией DDoS (staging) — 2 раза в год.

**Связанные требования:**
- `human/artifacts/requirements/REQ-CON.INFRA.deployment-geography.md` — DDoS protection по типам развёртывания
- `human/artifacts/requirements/REQ-FUN.INFRA.runbook-procedures.md` — DDoS Attack Response runbook

---

## REQ-SEC-80: Secret Scanning (pre-commit and CI)

### Инструменты
- **Pre-commit:** gitleaks (v8.18+) — обязателен для установки разработчиками (инструкция в `contributing.adoc`).
- **CI:** GitLab Secret Detection + gitleaks в pipeline — обязательная стадия перед merge.

### Пороги

| Тип секрета | Пример | Severity | Действие CI | Действие pre-commit |
|-------------|--------|----------|-------------|---------------------|
| Production ключи | `aws_secret_key = "AKIA..."` | **Block** | ❌ MR блокируется | ❌ commit отклоняется |
| Тестовые ключи | `test_api_key = "test_123"` | **Warning** | ⚠️ Warning, MR не блокируется | ⚠️ Предупреждение |
| Плейсхолдеры | `api_key = "<your-key>"` | **Ignore** | ✅ Игнорируется | ✅ Игнорируется |

### Исключения
Исключения задаются в `.gitleaks.toml` в корне репозитория. Изменения в этот файл требуют approval от Security Lead.

### Пример `.gitleaks.toml`

```toml
title = "VEDO Core secret scanner"

[extend]
useDefault = true

[[rules]]
id = "vedo-jwt-signing-key"
description = "VEDO Core JWT signing key"
regex = '''VEDO_JWT_SECRET\s*=\s*["']?[A-Za-z0-9+/=]{40,}'''
tags = ["key", "jwt"]

[allowlist]
description = "Test files and examples"
paths = ['''**/*_test\.go''', '''**/*_test\.rs''', '''**/examples/''']
regexes = ['''<your-[\w-]+>''']  # плейсхолдеры
```

### Secret Rotation

| Тип секрета | Срок ротации | Процедура |
|-------------|--------------|-----------|
| JWT signing keys | 90 дней | См. `vedo-cli rotate-secrets` |
| Service account tokens | 90 дней | Обновить в GitLab CI variables |
| Database passwords (production) | 180 дней | Через Vault / KMS |
| API-ключи интеграций | 180 дней | Обновить в конфигурации tenant |

**Экстренная ротация (при компрометации):**
1. Security Lead подтверждает компрометацию.
2. Отзыв секрета (revoke) — в течение 1 часа.
3. Генерация нового секрета и деплой.
4. Уведомление клиентов (если затронуты их данные).

**Ответственный:** Security Lead.

**Связанные артефакты:**
- `human/artifacts/requirements/REQ-FUN.PROCESS.preprod-release-gates.md` — Security Gate с порогами секретов
- `docs/antora/developer-guide/modules/ROOT/pages/contributing.adoc` — установка pre-commit hooks

---

## REQ-SEC-90: Verifiable Builds (SLSA Level 2)

**Описание:** Все production-артефакты должны быть криптографически подписаны и сопровождаться provenance (SLSA Level 2), чтобы обеспечить верификацию целостности и происхождения.

**Требования:**

| Элемент | Требование |
|---------|-------------|
| **Подпись** | Docker image подписан с помощью Sigstore (cosign). |
| **Provenance** | Содержит: source commit hash, builder ID (GitLab CI), список зависимостей (SBOM). |
| **Верификация** | Перед деплоем в production выполняется проверка подписи и provenance (`cosign verify` + `slsa-verifier`). |
| **Хранение** | Подписи и provenance хранятся в том же registry (как отдельные артефакты). |
| **Ротация ключей** | Подписывающий ключ ротируется ежегодно (Security Lead). |
| **Аудит** | Все подписанные артефакты имеют записи в audit log (кто подписал, когда, какой коммит). |

**Исключения:**

| Сценарий | Требование | Обоснование |
|----------|------------|-------------|
| Non-production (dev, staging) | Опционально | Упрощение отладки, риск ниже |
| On-premise развёртывание | Опционально | Заказчик доверяет своему registry |
| Air-gapped | Внутренняя PKI (не Sigstore) | Нет доступа к публичному Sigstore |
| Community / бесплатный тариф | Не требуется | Best-effort |

**Соответствие SLSA:** Level 2 (цель для MVP).

**Измерение соблюдения:**
- Метрика: % production-релизов, подписанных и верифицированных.
- Цель: 100% для всех релизов с тегом `v*.*.*`.

**Ответственный:** Security Lead + DevOps.

**Связанные артефакты:**
- `human/artifacts/requirements/REQ-NFR.SECURITY.sca-sbom-gating.md` — Verifiable Builds & SLSA Compliance
- `human/artifacts/requirements/REQ-FUN.PROCESS.preprod-release-gates.md` — Security Gate с SLSA/provenance
- `ADR-DES.PROCESS.gitlab-ci-cd-strategy` — Signing and Verification of Artifacts (adrs.md)

---

## REQ-SEC-100: License Compliance for Open-Source Dependencies

**Описание:** Все open-source зависимости, используемые в VEDO Core, должны соответствовать утверждённой лицензионной политике. Нарушения должны быть обнаружены автоматически в CI и заблокированы до релиза.

### Допустимые лицензии

Лицензии, разрешённые для использования в production-зависимостях (без ограничений):

| Лицензия | SPDX идентификатор | Примечание |
|----------|-------------------|------------|
| MIT | `MIT` | ✅ Разрешена |
| Apache License 2.0 | `Apache-2.0` | ✅ Разрешена |
| BSD 2-Clause | `BSD-2-Clause` | ✅ Разрешена |
| BSD 3-Clause | `BSD-3-Clause` | ✅ Разрешена |
| ISC | `ISC` | ✅ Разрешена |
| Mozilla Public License 2.0 | `MPL-2.0` | ✅ Разрешена (только для библиотек, не для кода VEDO) |
| Creative Commons Zero v1.0 | `CC0-1.0` | ✅ Разрешена (данные, не код) |
| The Unlicense | `Unlicense` | ✅ Разрешена |

### Запрещённые лицензии

Лицензии, **запрещённые** для использования в любых зависимостях (production и dev):

| Лицензия | SPDX идентификатор | Причина |
|----------|-------------------|---------|
| GNU General Public License v2.0 | `GPL-2.0` | Copyleft, требует раскрытия исходного кода |
| GNU General Public License v3.0 | `GPL-3.0` | Copyleft, требует раскрытия исходного кода |
| GNU Affero General Public License v3.0 | `AGPL-3.0` | Copyleft для сетевых сервисов (SaaS) — несовместимо с коммерческим SaaS |
| GNU Lesser General Public License v2.1 | `LGPL-2.1` | Copyleft, ограничения на линковку |
| GNU Lesser General Public License v3.0 | `LGPL-3.0` | Copyleft, ограничения на линковку |
| Server Side Public License | `SSPL` | Несовместимо с коммерческим SaaS (MongoDB license) |
| Creative Commons Non-Commercial | `CC-BY-NC-*` | Запрещает коммерческое использование |
| Неопределённые / custom | `LicenseRef-*` | Требует юридической проверки |

### Лицензии, требующие юридической проверки

При обнаружении следующих лицензий CI выдаёт **warning** и блокирует релиз до проверки юристом:

| Лицензия | SPDX идентификатор |
|----------|-------------------|
| Common Development and Distribution License | `CDDL-1.0` |
| Eclipse Public License 2.0 | `EPL-2.0` |
| JSON License | `JSON` |
| WTFPL | `WTFPL` |
| OpenSSL License | `OpenSSL` |

### Инструменты сканирования

| Экосистема | Инструмент | Команда |
|------------|------------|---------|
| Rust (cargo) | `cargo-deny` | `cargo deny check licenses` |
| Go (modules) | `go-licenses` | `go-licenses check ./...` |
| Python (pip) | `pip-licenses` | `pip-licenses --fail-on ...` |
| Node.js / TypeScript | `license-checker` | `license-checker --failOn "GPL;AGPL;LGPL;SSPL"` |
| Docker images | `trivy` | `trivy image --license-compliance` |

### Интеграция в CI

```yaml
# Пример для .gitlab-ci.yml
license-compliance:
  stage: security
  script:
    - cargo deny check licenses     # Rust
    - go-licenses check ./...      # Go
    - pip-licenses --fail-on GPL,AGPL,SSPL --allow-only MIT,Apache-2.0,BSD # Python
    - npx license-checker --failOn "GPL;AGPL;LGPL;SSPL" # Node.js
  rules:
    - if: $CI_MERGE_REQUEST_ID
    - if: $CI_COMMIT_TAG
  allow_failure: false
```

### Пороги

| Severity | Условие | Действие в CI | Действие для релиза |
|----------|---------|---------------|---------------------|
| **Block** | Обнаружена запрещённая лицензия (см. раздел "Запрещённые лицензии") | ❌ CI красный, MR блокируется | ❌ Релиз блокируется |
| **Review required** | Обнаружена лицензия, требующая проверки (раздел "Лицензии, требующие юридической проверки") | ⚠️ Warning, MR не блокируется | ⚠️ Релиз блокируется до юридической проверки |
| **Allow** | Допустимая лицензия (раздел "Допустимые лицензии") | ✅ PASS | ✅ Разрешён |

### Процедура обработки нарушений

1. **Обнаружение** (CI показывает нарушение).
2. **Юридическая оценка** (Security Lead + Legal):
   - Можно ли использовать библиотеку под этой лицензией?
   - Есть ли альтернатива с допустимой лицензией?
3. **Принятие решения:**
   - **Замена библиотеки** (предпочтительно) — создать issue на замену.
   - **Получение специального разрешения** (только для юрлица VEDO, с письменным approval).
4. **Документирование исключения** (в `licenses-exceptions.yaml`).
5. **Обновление CI** (добавить исключение в allowlist, если разрешено).

### Исключения (зависимости с проверенными licence-exceptions)

Исключения хранятся в файле `licenses-exceptions.yaml`:

```yaml
# licenses-exceptions.yaml
exceptions:
  - package: "openssl-sys"
    version: "0.9.0"
    license: "OpenSSL"
    reason: "No alternative for crypto operations, reviewed by Legal on 2026-05-23"
    approved_by: "Security Lead"
    expiry: "2027-05-23"
```

### Ответственность

| Роль | Ответственность |
|------|-----------------|
| **Security Lead** | Утверждение политики, юридическая оценка, ведение exceptions |
| **Legal / Compliance** | Финальное решение по неоднозначным лицензиям |
| **DevOps** | Настройка CI-джоб для license scanning |
| **Разработчик** | Реагирование на найденные нарушения (замена библиотеки) |

**Связанные артефакты:**
- `human/artifacts/requirements/REQ-NFR.SECURITY.sca-sbom-gating.md` — License Compliance (дополнение к SCA)
- `human/artifacts/requirements/REQ-FUN.PROCESS.preprod-release-gates.md` — Security Gate с License Compliance
- `docs/antora/developer-guide/modules/ROOT/pages/contributing.adoc` — инструкция для разработчиков по добавлению зависимостей
- `licenses-exceptions.yaml` — реестр исключений
