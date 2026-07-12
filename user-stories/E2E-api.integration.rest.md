<a id="e2e-api.integration.rest"></a>
# E2E-api.integration.rest: Интеграция через REST API

```gherkin
@E2E-api.integration.rest @e2e @api @P1
Feature: E2E-api.integration.rest Интеграция через REST API

  Scenario: Внешняя система создаёт онтологию и подписывается на изменения
    Given внешняя система имеет API-ключ с правами "Editor"
    When система отправляет POST-запрос на "/api/v1/ontologies"
    And тело запроса содержит name "ProductCatalog"
    Then API возвращает HTTP 201 Created
    And онтология "ProductCatalog" создана
    When система настраивает webhook на URL "https://webhook.site/vedo-events"
    And подписывается на события "class.created" и "class.updated"
    And создаёт класс "Product" через REST API
    Then webhook-сервер получает уведомление о создании класса
    When система экспортирует онтологию через REST API в формате Turtle
    Then ответ содержит Turtle-сериализацию онтологии
```
