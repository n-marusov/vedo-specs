<a id="us-support.tickets.create-track"></a>
# US-support.tickets.create-track: Создание и отслеживание тикета поддержки

```gherkin
@US-support.tickets.create-track @UC-support.tickets.create-and-track-support-ticket @P0 @support @tickets
Feature: US-support.tickets.create-track Создание и отслеживание тикета поддержки

  Background:
    Given пользователь аутентифицирован в VEDO Core

  Scenario: Пользователь создает тикет и отслеживает его статус
    When пользователь открывает форму поддержки
    And заполняет тему, описание, категорию и severity
    And отправляет тикет
    Then система создает тикет в статусе "New"
    And система автоматически прикладывает технические метаданные
    And пользователь видит изменение статусов тикета в истории
```
