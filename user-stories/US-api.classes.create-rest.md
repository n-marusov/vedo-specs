<a id="us-api.classes.create-rest"></a>
# US-api.classes.create-rest: Создание класса через REST API

```gherkin
@US-api.classes.create-rest @UC-api.integration.integrate-through-platform-apis @P1 @api @rest @integration
Feature: US-api.classes.create-rest Создание класса через REST API

  Background:
    Given разработчик имеет доступ к REST API VEDO
    And открыта онтология "test"

  Scenario: Разработчик создаёт класс через REST API
    Given API-ключ имеет права "Editor"
    When разработчик отправляет POST-запрос на "/api/v1/ontologies/test/classes"
    And тело запроса содержит label "Vehicle"
    And тело запроса содержит parents "owl:Thing"
    Then API возвращает HTTP 201 Created
    And тело ответа содержит id "Vehicle"
    And класс "Vehicle" создан в онтологии

  Scenario: API отклоняет запрос без аутентификации
    Given API-запрос не содержит JWT-токен
    When разработчик отправляет POST-запрос на "/api/v1/ontologies/test/classes"
    Then API возвращает HTTP 401 Unauthorized
    And тело ответа содержит "Authentication required"
```
