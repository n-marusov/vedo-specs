<a id="us-api.webhooks.manage"></a>
# US-api.webhooks.manage: Управление webhook-подписками и доставка событий

```gherkin
@US-api.webhooks.manage @UC-api.webhooks.manage-subscriptions-and-delivery @P1 @api @webhook
Feature: US-api.webhooks.manage Управление webhook-подписками и доставка событий

  Background:
    Given разработчик аутентифицирован в API

  Scenario: Разработчик получает webhook при событии изменения
    When разработчик создает webhook-подписку на событие "class.updated"
    And в онтологии происходит обновление класса
    Then система отправляет HTTP POST на подписанный endpoint
    And фиксирует результат доставки
```
