<a id="us-api.docs.openapi"></a>
# US-api.docs.openapi: Использование OpenAPI/Swagger для тестирования API

```gherkin
@US-api.docs.openapi @UC-api.docs.view-openapi-specification @P0 @api @docs
Feature: US-api.docs.openapi Использование OpenAPI/Swagger для тестирования API

  Background:
    Given опубликована актуальная OpenAPI спецификация

  Scenario: Интегратор находит endpoint и выполняет тестовый вызов
    When интегратор открывает Swagger UI
    And выбирает endpoint создания класса
    And отправляет тестовый запрос
    Then система показывает структуру запроса и ответа
    And тестовый вызов выполняется по спецификации
```
