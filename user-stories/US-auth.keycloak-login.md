<a id="us-auth.keycloak-login"></a>
# US-auth.keycloak-login: Аутентификация через Keycloak по логину и паролю

```gherkin
@US-auth.keycloak-login @UC-auth.keycloak-oidc-login @P0 @auth @security
Feature: US-auth.keycloak-login Аутентификация через Keycloak по логину и паролю

  Background:
    Given Keycloak запущен на localhost:8180
    And realm "vedo-core" импортирован с пользователями (alice, bob, carol, dave, eve, frank, system-bot)

  Scenario: Неаутентифицированный пользователь перенаправляется на страницу входа
    When пользователь открывает защищённый маршрут "/dashboard"
    Then система перенаправляет на "/login"
    And отображаются кнопки OAuth-провайдеров

  Scenario: Пользователь входит через Corporate SSO с валидными учётными данными
    Given пользователь находится на странице "/login"
    When пользователь нажимает "Corporate SSO"
    And система перенаправляет на страницу входа Keycloak
    And пользователь вводит логин и пароль
    Then Keycloak аутентифицирует пользователя
    And система перенаправляет обратно в приложение через OIDC callback
    And пользователь видит дашборд

  Scenario: Пользователь получает правильную роль после входа
    Given пользователь "frank" входит через Corporate SSO
    Then сессия содержит роль "owner"
    Given пользователь "eve" входит через Corporate SSO
    Then сессия содержит роль "admin"
    Given пользователь "alice" входит через Corporate SSO
    Then сессия содержит роль "viewer"
    Given пользователь "bob" входит через Corporate SSO
    Then сессия содержит роль "editor"

  Scenario: Система отклоняет вход с неверным паролем
    When пользователь вводит неверный пароль на странице Keycloak
    Then Keycloak показывает сообщение об ошибке
    And пользователь остаётся на странице входа Keycloak
    And редирект в приложение не происходит
```
