# Регрессионный контроль совпадения JWT-ключей в test-окружении

| Атрибут | Значение |
|---------|----------|
| **ID** | REQ-NFR.SECURITY.jwt-key-mismatch |
| **Уровень** | NFR |
| **Атрибут качества** | Functionality |
| **Приоритет** | P0 |
| **Статус** | УТВЕРЖДЕНО |
| **Источник** | Bug: 401 JWT key mismatch в docker-compose.test.yml |
| **Критерии приёмки** | E2E-тест `tests/e2e/playwright/tests/api/rest/jwt-auth.spec.ts` проходит без 401 для всех test-JWT (OWNER/VIEWER/EDITOR) |

---

## Назначение

Зафиксировать недопустимость регрессии дефекта, при котором self-signed test JWT-токены проверялисьagainst Keycloak JWKS вместо тестового RSA public key (`JWT_DEV_PUBLIC_KEY_PEM`), настроенного на API Gateway в test-окружении.

## Контекст дефекта

В `deploy/docker-compose.test.yml` API Gateway настраивается парой ключей:

- `JWT_DEV_PUBLIC_KEY_PEM` — RSA public key, которым проверяются подписи test-токенов;
- `test-jwt-key.pem` — соответствующий private key, которым подписываются токены в `tests/e2e/playwright/tests/jwt-tokens.ts`.

Дефект проявлялся, когда gateway fallback-ил на Keycloak JWKS endpoint (production-режим валидации) при невалидной/отсутствующей конфигурации `JWT_DEV_PUBLIC_KEY_PEM`. В результате все test-токены возвращали `401 Unauthorized`, и весь E2E-набор становился зелёным только через моки, скрывая регрессии авторизации.

## Требование

API Gateway в test-окружении **обязан** принимать self-signed test JWT-токены, подписанные test-private-key, при условии что `JWT_DEV_PUBLIC_KEY_PEM` сконфигурирован. Валидация против Keycloak JWKS в test-окружении **запрещена**, если только тест явно не проверяет production-сценарий Keycloak.

## Критерии приёмки

| # | Условие | Проверка |
|---|---------|----------|
| 1 | OWNER_JWT принимается (HTTP < 500, ≠ 401) | `jwt-auth.spec.ts` → `[FIX] OWNER_JWT should be accepted` |
| 2 | VIEWER_JWT принимается для read-операций | `jwt-auth.spec.ts` → `[FIX] VIEWER_JWT should be accepted for read operations` |
| 3 | EDITOR_JWT принимается | `jwt-auth.spec.ts` → `[FIX] EDITOR_JWT should be accepted` |
| 4 | Токены содержат ожидаемые claims (`sub`, `user_id`, `organization_id`, `roles`) | `jwt-auth.spec.ts` → Token claims tests |
| 5 | Запросы без/с некорректным Authorization возвращают 401 | `jwt-auth.spec.ts` → Auth rejection tests |

## Недопустимые обходные пути

- Отключение JWT-валидации на gateway для прохождения тестов.
- Использование production Keycloak-токенов в E2E (маскирует конфигурационный drift).
- Мокирование auth-слоя в тех тестах, которые претендуют на проверку end-to-end контракта.

## Артефакты

- Тест: `tests/e2e/playwright/tests/api/rest/jwt-auth.spec.ts`
- Токены: `tests/e2e/playwright/tests/jwt-tokens.ts`
- Конфигурация: `deploy/docker-compose.test.yml` (env: `JWT_DEV_PUBLIC_KEY_PEM`)
- Traceability: `base:req/REQ-NFR.SECURITY.jwt-key-mismatch` в `.ai-factory/traceability/traceability.ttl`
