# CI/CD Requirements

| Атрибут | Значение |
|---------|----------|
| **ID** | REQ-FUN.PROCESS.ci-cd-requirements |
| **Уровень** | FUN |
| **Атрибут качества** | Implementation |
| **Приоритет** | P0 |
| **Статус** | УТВЕРЖДЕНО |
| **Источник** | — |
| **Критерии приёмки** | — |

---

## CI/CD Secrets Management

### REQ-CICD-01: Vault Integration for CI

Для CI/CD (GitLab CI) рекомендуется использовать HashiCorp Vault для хранения токенов `vedo-cli` и других секретов.

**Пример использования в `.gitlab-ci.yml`:**

```yaml
deploy:
  script:
    - vedo-cli backup create
  secrets:
    VAULT_TOKEN:
      vault: secret/vedo/ci/token@secrets-engine
```

### REQ-CICD-02: Fallback to CI Variables

Если Vault недоступен в CI, допускается использование masked CI variables (например, `$VEDO_CLI_TOKEN`).

### REQ-CICD-03: Local Development (без Vault)

Vault **не входит** в `docker-compose.yaml` на milestone 001. Для локальной разработки токены и секреты хранятся в `~/.vedo/config.yaml` или передаются через переменные окружения.

Цепочка разрешения credentials для local dev:

1. Переменные окружения (`VEDO_CLI_TOKEN`).
2. Файл `~/.vedo/config.yaml`.

Vault и AWS Secrets Manager из цепочки исключены. Полная цепочка (Vault → AWS Secrets Manager → env → plain-text) применяется только в production/CI.

| Окружение | Источник credentials | Vault в compose? |
|-----------|---------------------|------------------|
| Локальная разработка | `~/.vedo/config.yaml` / env | ❌ Нет |
| CI/CD | Vault (`secrets:`) / CI variables | N/A |
| Production | Vault (отдельный кластер) | N/A |
| On-premise | Опционально (Vault или файл) | ❌ (по умолчанию) |

### REQ-CICD-04: No Secrets in Repositories

Секреты не должны храниться в Git-репозитории. Файл `~/.vedo/config.yaml` находится за пределами репозитория. CI variables используют masked и protected флаги.

---

## CI/CD Infrastructure Requirements

### CI/CD Runner Resources

| Параметр | Минимальное требование | Рекомендованное требование | Обоснование |
|----------|------------------------|----------------------------|-------------|
| **CPU** | 4 vCPU | 8 vCPU | Сборка Rust/Go кода требует значительных ресурсов |
| **RAM** | 8 GB | 16 GB | Для параллельных сборок и тестов (особенно Python + Rust) |
| **Диск (SSD)** | 50 GB | 100 GB | Кэш зависимостей (cargo, go mod, npm, pip), артефакты сборки |
| **Тип runner** | Kubernetes executor (динамический) | Kubernetes + spot instances (cost optimization) | Масштабирование под нагрузку |

**SLO для CI/CD (доступность GitLab CI):**

| Параметр | Целевое значение | Измерение |
|----------|------------------|-----------|
| **Доступность CI** | 99.5% (ежемесячно) | Успешные вызовы GitLab API для создания pipeline |
| **Максимальное время ожидания runner** | ≤ 5 минут (p95) | Время от триггера до старта job |
| **MTTR (при недоступности CI)** | ≤ 4 часа | Ручное восстановление (SRE on-call) |

**Capacity planning:**
- Каждый активный разработчик генерирует ~20 pipeline в день (MR + коммиты).
- Пиковая нагрузка: 10 параллельных сборок (10 runner pods).
- Рекомендуемый пул runner: 10–20 pods (автомасштабирование).

### Политика версий build-инструментов

**Принцип:** Все build-инструменты должны быть pinned до конкретных версий в CI (как в локальной разработке, так и в pipeline).

| Инструмент | Поддерживаемая версия | Grace period после выхода новой мажорной версии | Обновление |
|------------|----------------------|------------------------------------------------|------------|
| **Rust** | 1.85.x | 90 дней | Ежеквартально (автоматизированный PR через Renovate) |
| **Go** | 1.23.x | 90 дней | Ежеквартально (автообновление) |
| **Python** | 3.12.x | 90 дней | Ежеквартально (автообновление) |
| **Node.js** | 20.x LTS | 90 дней | Ежеквартально (автообновление) |
| **cargo-audit** | Согласно `REQ-SEC-50` | 14 дней | Еженедельно (security patches) |

**Процедура обновления:**
1. Renovate создаёт PR с обновлением версии в CI (`.gitlab-ci.yml`, `Dockerfile`, `rust-toolchain.toml`).
2. CI проверяет совместимость (все тесты проходят).
3. Tech Lead утверждает PR.
4. В течение grace period (90 дней) старая версия всё ещё поддерживается.
5. После grace period — PR с обновлением мажорной версии обязателен для прохождения CI.

**Исключения:**
- Security patches — обновляются немедленно (без grace period).
- Breaking changes в инструментах (например, Rust edition) — требуют отдельного планирования.

**Пример фиксации версий в CI:**

```yaml
# .gitlab-ci.yml
variables:
  RUST_VERSION: "1.85.0"
  GO_VERSION: "1.23.0"
  PYTHON_VERSION: "3.12.0"
  NODE_VERSION: "20.11.0"
```

### CI/CD Maintenance

| Действие | Частота | Ответственный |
|----------|---------|---------------|
| Проверка доступности CI (SLO) | Ежедневно | SRE on-call |
| Обновление runner образов (патчи) | Еженедельно | DevOps |
| Очистка старых артефактов (> 30 дней) | Еженедельно | DevOps (автоматизировано) |
| Обновление build-инструментов (Renovate PR) | Еженедельно | Tech Lead / DevOps |

## Caching Strategy for CI/CD

Для ускорения сборок и снижения нагрузки на registry используется кэширование зависимостей.

| Экосистема | Что кэшируется | Размер кэша | TTL кэша | Хранилище |
|------------|----------------|-------------|----------|-----------|
| Rust (cargo) | `~/.cargo/registry`, `target/` | до 5 GB | 7 дней | GitLab CI cache (S3) |
| Go | `~/go/pkg/mod` | до 2 GB | 7 дней | GitLab CI cache |
| Python | `~/.cache/pip` | до 1 GB | 7 дней | GitLab CI cache |
| Node.js | `node_modules/` | до 500 MB | 7 дней | GitLab CI cache |
| Docker | слои images (registry) | неограниченно | постоянно | GitLab Container Registry |

**Политика очистки:**
- Caches старше 7 дней удаляются автоматически (GitLab policy).
- Для релизных тегов кэш не используется (сборка с нуля для воспроизводимости).

**Cache size monitoring:**
- Метрика: `ci_cache_size_bytes` (Prometheus).
- Алерт: при превышении 10 GB (предупреждение), 20 GB (критическое — требует ручной очистки).

## REQ-CICD-10: SCA Toolchain Version Control

Требования к управлению версиями SCA-инструментов в CI/CD:

| Требование | Проверка |
|------------|----------|
| Все SCA-инструменты должны быть pinned до конкретной версии | CI проверяет `--version` перед выполнением |
| Обновление версий — через automated PR (Renovate/Dependabot) | Настройка Renovate (см. `renovate.json`) |
| Security patches (critical CVE в инструменте) — обновление в течение 24 часов | Процедура оповещения security@vedo |
| Мажорные версии — тестирование в staging перед обновлением | Отдельная задача в бэклоге |

**Ответственный:** DevOps / Security Lead.

### Пример SCA-джоб с pinned версиями

```yaml
# Пример конфигурации SCA-джоб с pinned версиями (milestone 001+)

variables:
  CARGO_AUDIT_VERSION: "0.21.2"
  GOVULNCHECK_VERSION: "1.0.4"
  SAFETY_VERSION: "3.4.0"
  TRIVY_VERSION: "0.69.6"

security-scan-rust:
  stage: security
  image: rust:1.85
  before_script:
    - cargo install cargo-audit --version $CARGO_AUDIT_VERSION
    - cargo audit --version  # проверка версии
  script:
    - cargo audit --deny warnings
  rules:
    - if: $CI_MERGE_REQUEST_ID
    - if: $CI_COMMIT_TAG

security-scan-go:
  stage: security
  image: golang:1.23
  before_script:
    - go install golang.org/x/vuln/cmd/govulncheck@v$GOVULNCHECK_VERSION
    - govulncheck -version  # проверка версии
  script:
    - govulncheck ./...
  rules:
    - if: $CI_MERGE_REQUEST_ID
    - if: $CI_COMMIT_TAG

security-scan-python:
  stage: security
  image: python:3.12
  before_script:
    - pip install safety==$SAFETY_VERSION
    - safety --version  # проверка версии
  script:
    - safety scan --json --output json
  rules:
    - if: $CI_MERGE_REQUEST_ID
    - if: $CI_COMMIT_TAG

security-scan-container:
  stage: security
  image: aquasec/trivy:$TRIVY_VERSION
  before_script:
    - trivy --version  # проверка версии
  script:
    - trivy image --severity HIGH,CRITICAL --exit-code 1 $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA
  rules:
    - if: $CI_COMMIT_TAG
```
