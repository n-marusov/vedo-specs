<a id="us-support.status.incident"></a>
# US-support.status.incident: Просмотр публичной статус-страницы при инциденте

```gherkin
@US-support.status.incident @UC-support.status.view-public-status-page @P1 @support @status
Feature: US-support.status.incident Просмотр публичной статус-страницы при инциденте

  Background:
    Given подтвержден инцидент P1

  Scenario: Пользователь получает своевременные апдейты инцидента
    When система публикует инцидент на status page
    Then первый публичный апдейт опубликован не позже 10 минут
    And сообщение содержит incident ID, затронутые компоненты и planned update time
    And последующие обновления публикуются в пределах SLA
```
