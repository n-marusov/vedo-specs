# ADR-DES.SECURITY.cli-mfa-strategy

**Дата:** 2026-05-19  
**Статус:** PROPOSED → ACCEPTED

## Контекст

`ADR-DES.SECURITY.mfa-critical-ops-mandate` определяет категории операций, требующих MFA:

| Категория | Операции | Политика MFA |
|-----------|----------|--------------|
| **A** | `backup delete`, `tenant purge`, `secure erase`, `decommission --purge-data` | MFA обязателен, нельзя отключить |
| **B** | `production restore`, `migration rollback` | MFA обязателен, нельзя отключить |
| **C** | `ontology delete`, `account close`, `branch delete` | MFA включён по умолчанию, может настраиваться |
| **D** | `export`, `diagnose`, `backup create` | MFA не требуется |

Однако ADR не специфицирует, как именно MFA реализуется для CLI-утилиты `vedo-cli` — среды без браузера, где пользователь вводит команды в терминале, а CI/CD должен работать полностью неинтерактивно.

Ключевые противоречия:
- CLI не имеет браузера для OIDC authorization code flow (как UI).
- Привилегированные операции (A/B) требуют MFA при каждом вызове — нельзя кэшировать на сессию.
- CI/CD не может ввести TOTP-код — требуется service account без MFA.
- Keycloak уже принят как единый IdP для VEDO Core (`ADR-IMPL.STACK.auth-service-go-strategy`, C4-контекстная диаграмма).
- `ADR-DES.SECURITY.break-glass-access-strategy` определяет Emergency Admin L1-L3 для сценариев, когда Keycloak недоступен.

## Требование-источник

- `ADR-DES.SECURITY.mfa-critical-ops-mandate` — категоризация операций и политика MFA
- `ADR-DES.INFRA.vedo-cli-admin-boundary` — роль vedo-cli в экосистеме
- `ADR-DES.SECURITY.break-glass-access-strategy` — Emergency Admin при отказе IdP
- `human/artifacts/requirements/REQ-FUN.INFRA.vedo-cli-specification.md` — требования к CLI

## Решение

Использовать **Keycloak как единый IdP** для MFA в `vedo-cli`, реализовав два режима аутентификации:

**1. Интерактивный режим (администратор в терминале):**

```
$ vedo-cli backup create --full
Enter username: admin@vedo.local
Enter password: ********
Enter TOTP code: 123456
✅ Authenticated (token valid for 1h)
← refresh_token сохранён в ~/.vedo/config.yaml
```

- CLI выполняет OIDC Resource Owner Password Grant (ROPG) + TOTP claim в адрес Keycloak.
- После успешной аутентификации CLI получает `access_token` с коротким TTL (1 час) и `refresh_token`.
- `refresh_token` кэшируется в `~/.vedo/config.yaml` (зашифрованный файл).
- Для команд категорий A/B MFA-сессия не кэшируется — каждый вызов требует нового TOTP (даже если `access_token` действителен).
- Для команд категорий C TOTP запрашивается 1 раз за время жизни refresh_token.

**2. Неинтерактивный режим (CI/CD, automation):**

```bash
$ vedo-cli backup create --token $SERVICE_ACCOUNT_TOKEN
```

- Service account в Keycloak с `client_credentials` grant.
- Service account не имеет MFA, но scope ограничен предопределённым набором операций (категория D, read-only, backup create).
- Операции категорий A/B через service account запрещены на уровне Keycloak client scopes.
- Audit логирует `actor=service-account:<client_id>` для отличения автоматизированных операций от человеческих.

**3. Break-glass (Keycloak недоступен):**

- Используется L1 Emergency Admin из `ADR-DES.SECURITY.break-glass-access-strategy` через endpoint `/emergency/login`.
- Emergency Admin обходит MFA (компенсируется Shamir splitting, immutable audit и уведомлениями).

## Рассмотренные альтернативы

**Альтернатива A: Встроенный TOTP в CLI (без Keycloak)**

Реализовать генерацию и верификацию TOTP-кодов непосредственно в CLI, хранить seed в локальном конфигурационном файле.

| Критерий | Keycloak | Встроенный TOTP |
|----------|----------|-----------------|
| Единый каталог пользователей | ✅ (один для UI и CLI) | ❌ (дублирование) |
| Self-service MFA | ✅ (через UI Keycloak) | ❌ (требуется отдельный CLI-интерфейс) |
| Резервные коды (recovery) | ✅ (встроенные в Keycloak) | ❌ (требуется реализация) |
| M2M-токены (service accounts) | ✅ (client_credentials) | ❌ (нет) |
| Централизованный аудит | ✅ (Keycloak event log) | ⚠️ (требуется своя реализация) |
| Зависимость от Keycloak | ✅ (требуется доступность) | ❌ (независим) |

**Решение:** Отклонено. Дублирование каталога пользователей и MFA-настроек создаёт эксплуатационные издержки и риск рассинхронизации. Отсутствие M2M-токенов не позволяет реализовать CI/CD-сценарий без хранения статических токенов.

**Альтернатива B: Только пароль для CLI, MFA только для UI**

Администратор аутентифицируется в CLI по паролю, MFA требуется только при входе в веб-интерфейс.

| Критерий | Keycloak CLI MFA | Только пароль в CLI |
|----------|------------------|---------------------|
| Соответствие `mfa-critical-ops-mandate` | ✅ | ❌ (нарушает mandate) |
| Защита разрушительных операций | ✅ | ❌ (один фактор) |
| Единая политика безопасности | ✅ | ❌ (разные политики для UI и CLI) |

**Решение:** Отклонено. Противоречит `ADR-DES.SECURITY.mfa-critical-ops-mandate`, который требует MFA для категорий A/B независимо от канала (UI или CLI).

**Альтернатива C: Аппаратные токены (YubiKey) без Keycloak**

Использовать FIDO2/WebAuthn напрямую через YubiKey, без центрального IdP.

| Критерий | Keycloak | YubiKey-only |
|----------|----------|--------------|
| Управление пользователями | ✅ централизованное | ❌ ручное на каждом устройстве |
| Резервные коды | ✅ встроенные | ❌ нет |
| Service accounts | ✅ client_credentials | ❌ нет |
| Стоимость внедрения | Средняя (интеграция) | Высокая (закупка токенов, настройка) |
| Air-gapped совместимость | ✅ (через break-glass) | ✅ |

**Решение:** Отклонено. Избыточно для MVP — YubiKey добавляет аппаратные затраты и не решает проблему service accounts для CI/CD. Break-glass уже покрывает сценарий недоступности Keycloak.

## Последствия

**Положительные:**

- Единый каталог пользователей: один источник правды для UI и CLI (Keycloak).
- Self-service MFA: администратор настраивает TOTP через веб-интерфейс Keycloak без участия DevOps.
- Резервные коды (recovery codes): встроенная функция Keycloak.
- Service accounts: CI/CD автоматизация через `client_credentials` grant без MFA, но с ограниченным scope.
- Централизованный аудит: Keycloak Event Listener фиксирует все попытки аутентификации (успех, неудача, использованный фактор).
- Break-glass согласован: Emergency Admin обходит MFA через независимый путь (Shamir splitting компенсирует отсутствие второго фактора).

**Отрицательные:**

- CLI зависит от Keycloak: если Keycloak недоступен, администратор не может выполнить привилегированные операции через CLI.
- ROPG grant: требует отправки пароля в теле запроса (в отличие от authorization code flow в браузере). Снижается трёхстороннее разделение секретов.
- Интерактивный ввод TOTP: администратор должен ввести TOTP-код в терминале, что менее удобно, чем браузерный flow.
- Дополнительный компонент: `AuthSessionManager` в vedo-cli для управления токенами и их кэширования.

**Меры снижения рисков:**

- Break-glass Emergency Admin (L1-L3) из `ADR-DES.SECURITY.break-glass-access-strategy` покрывает отказ Keycloak.
- `access_token` имеет короткий TTL (1 час), `refresh_token` кэшируется в зашифрованном виде в `~/.vedo/config.yaml`.
- Для категорий A/B MFA не кэшируется даже при живом `access_token` — каждый вызов требует нового TOTP (это соответствует `mfa-critical-ops-mandate`: «MFA не кэшируется; каждый вызов CLI требует нового TOTP/WebAuthn подтверждения»).
- Пароль не выводится на экран (term read password mode), не логируется, не сохраняется в shell history.
- ROPG grant допустим для trusted CLI-клиента (public client в Keycloak), но используется только с TOTP claim — пароль без TOTP недостаточен для операций A/B/C.
- Аудит аутентификации через Keycloak events + отдельное событие в CLI audit trail.

## Детали реализации

**Пример кэширования refresh_token в `~/.vedo/config.yaml`:**

```yaml
auth:
  realm: vedo-core
  client_id: vedo-cli
  refresh_token: eyJhbGciOiJSUzI1NiIs...  # зашифрован мастер-ключом из credential chain
  token_endpoint: https://keycloak.vedo.local/realms/vedo-core/protocol/openid-connect/token
```

**Пример OIDC ROPG + TOTP в Go (иллюстративный):**

```go
type AuthSessionManager struct {
    tokenEndpoint string
    clientID      string
    httpClient    *http.Client
}

func (m *AuthSessionManager) Login(username, password, totpCode string) (*TokenSet, error) {
    form := url.Values{
        "client_id":  {m.clientID},
        "grant_type": {"password"},
        "username":   {username},
        "password":   {password},
        "totp":       {totpCode},
    }
    resp, err := m.httpClient.PostForm(m.tokenEndpoint, form)
    // → access_token, refresh_token, expires_in
}
```

**Пример получения service account токена (CI/CD):**

```bash
$ export VEDO_CLI_TOKEN=$(curl -s -X POST https://keycloak.vedo.local/realms/vedo-core/protocol/openid-connect/token \
  -d "client_id=vedo-cli-service" \
  -d "client_secret=$SERVICE_ACCOUNT_SECRET" \
  -d "grant_type=client_credentials" | jq -r '.access_token')

$ vedo-cli backup create --token $VEDO_CLI_TOKEN
```

## Связанные ADR

- `ADR-DES.SECURITY.mfa-critical-ops-mandate` — категоризация операций и политика MFA
- `ADR-DES.INFRA.vedo-cli-admin-boundary` — роль vedo-cli в экосистеме VEDO Core
- `ADR-DES.SECURITY.break-glass-access-strategy` — Emergency Admin при отказе IdP
- `ADR-IMPL.STACK.auth-service-go-strategy` — Keycloak интеграция в Auth Service
- `ADR-IMPL.STACK.vedo-cli-language-strategy` — Go как язык реализации vedo-cli
- `ADR-IMPL.SECURITY.vedo-cli-credentials-strategy` — цепочка получения credentials для vedo-cli

---
