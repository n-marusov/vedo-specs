<a id="us-api.auth.jwt"></a>
# US-api.auth.jwt: Получение и проверка JWT для вызова API

```gherkin
@US-api.auth.jwt @UC-api.auth.issue-and-validate-jwt-tokens @P0 @api @auth
Feature: US-api.auth.jwt Получение и проверка JWT для вызова API

  Background:
    Given разработчик имеет учетные данные в Keycloak

  Scenario: API Gateway принимает запрос с валидным JWT
    When разработчик получает JWT в Keycloak
    And отправляет запрос к API Gateway с Bearer-токеном
    Then API Gateway валидирует токен
    And маршрутизирует запрос к целевому сервису

  Scenario: API Gateway отклоняет запрос с невалидным JWT
    When разработчик отправляет запрос с просроченным токеном
    Then API Gateway возвращает ошибку авторизации
    And запрос не передается во внутренние сервисы
```
