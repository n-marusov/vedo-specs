# ADR-DES.PROCESS.gitlab-ci-cd-strategy

**Дата:** 2026-05-11  
**Статус:** Принято

## Контекст

VEDO Core состоит из нескольких сервисов и языков: Rust, Go, Python, TypeScript, документации на Antora и Docker/Helm развёртывания. Нужна единая процедура CI/CD, которая поставляется вместе с продуктом, автоматически проверяет качество при push/MR, собирает документацию и обеспечивает контролируемое развертывание в dev, staging и production.

## Требование-источник
- [ci-cd-procedures.md](requirements/REQ-FUN.PROCESS.ci-cd-procedures.md)

## Решение

Использовать GitLab CI как CI/CD платформу по умолчанию, поставляя `.gitlab-ci.yml` с обязательными стадиями: lint, test, build, docs, deploy-dev (автоматический), deploy-staging (ручной) и deploy-prod (ручной).

GitLab CI объединяет репозиторий, Container Registry, environments и manual approvals в единой корпоративной поставке, обеспечивая воспроизводимые проверки для всех языков стека (Rust, Go, Python, TypeScript), сборку документации Antora и контролируемое развёртывание с ручными gates для staging и production.

Production развёртывание и откат выполнять с формированием неизменяемого доказательства развёртывания; заказчики без GitLab адаптируют эталонный конвейер под свою систему; параметры развёртывания вынести в variables и Helm values.

## Рассмотренные альтернативы

| Альтернатива | Причина отклонения |
|--------------|--------------------|
| GitHub Actions | Хорош для open-source рабочих процессов, но GitLab CI лучше объединяет repository, registry, environments, manual approvals и корпоративный self-hosting в одной поставке |
| Jenkins | Гибкий, но требует больше инфраструктурной поддержки, plugin maintenance и ручной настройки |
| Только локальные scripts без CI/CD | Не обеспечивает повторяемые проверки, audit trail и контролируемое развёртывание |
| Разные CI/CD платформы для разных заказчиков | Увеличивает стоимость сопровождения и снижает воспроизводимость конвейера доставки |

## Signing and Verification of Artifacts (дополнение)

**Decision:** Все production-артефакты (Docker images) подписываются с помощью Sigstore (cosign) на уровне SLSA Level 2. Верификация подписи и provenance обязательна перед деплоем в production.

**Реализация в .gitlab-ci.yml:**

```yaml
variables:
  COSIGN_EXPERIMENTAL: "true"  # для OIDC интеграции

sign-image:
  stage: sign
  image: cgr.dev/chainguard/cosign:latest
  script:
    # Подпись image
    - cosign sign --key k8s://vedo-signing-secret $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA
    # Создание и подпись provenance
    - cosign attest --predicate provenance.json --key k8s://vedo-signing-secret $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA
  rules:
    - if: $CI_COMMIT_TAG
  needs: ["build-image"]

verify-image:
  stage: deploy
  image: cgr.dev/chainguard/cosign:latest
  before_script:
    - apt-get update && apt-get install -y wget
    - wget -O /usr/local/bin/slsa-verifier https://github.com/slsa-framework/slsa-verifier/releases/latest/download/slsa-verifier-linux-amd64
    - chmod +x /usr/local/bin/slsa-verifier
  script:
    # Проверка подписи
    - cosign verify --key vedo-pubkey.pem $CI_REGISTRY_IMAGE:$CI_COMMIT_TAG
    # Проверка provenance
    - slsa-verifier verify-image $CI_REGISTRY_IMAGE:$CI_COMMIT_TAG \
        --source-uri $CI_PROJECT_URL \
        --source-tag $CI_COMMIT_TAG
  rules:
    - if: $CI_COMMIT_TAG
  needs: ["sign-image"]
```

**Управление ключами:**
- Приватный ключ подписи хранится в GitLab CI variable `COSIGN_PRIVATE_KEY` (зашифрован).
- Публичный ключ (`vedo-pubkey.pem`) хранится в репозитории и распространяется с документацией (для on-premise верификации).
- Ротация ключей — раз в год (Security Lead).
- При компрометации ключа — экстренная ротация в течение 24 часов.

**Соответствие SLSA:** Level 2 (provenance подписан, build изолирован).

## Последствия

**Положительные последствия:**
- Единая поставляемая CI/CD конфигурация для всех языков и сервисов.
- GitLab Container Registry и GitLab environments упрощают Docker build/deploy flow.
- Ручные шлюзы staging/prod снижают риск случайного развёртывания в production.
- Сборка документации Antora становится частью конвейера релизов.
- Проверки качества документации становятся частью конвейера релизов.
- Customers могут адаптировать `.gitlab-ci.yml` под свои namespaces и окружения.
- Подпись и верификация артефактов (SLSA Level 2) обеспечивают доказуемую цепочку поставки.

**Отрицательные последствия:**
- Заказчики без GitLab должны адаптировать конвейер под свою CI/CD систему.
- Конвейер должен обслуживать несколько языковых экосистем и toolchains.
- Docker-in-Docker и Helm deploy требуют runner permissions и настройки Kubernetes credentials.
- Добавляются стадии `sign` и `verify`, увеличивающие время пайплайна.
- Управление ключами подписи (ротация, хранение, компрометация) добавляет operational overhead.

**Меры снижения рисков:**
- Держать `.gitlab-ci.yml` как reference implementation.
- Документировать required variables, registry credentials и Kubernetes contexts.
- Разделить конвейер на stages и jobs, чтобы заказчики могли отключать или заменять отдельные части.
- Вынести параметры развертывания в variables и Helm values.
- Автоматизировать ротацию ключей через `vedo-cli rotate-secrets`.
- Для on-premise/air-gapped предусмотреть внутреннюю PKI вместо Sigstore.

---
