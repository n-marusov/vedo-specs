<a id="us-git.commits.create"></a>
# US-git.commits.create: Фиксация изменений с сообщением коммита

```gherkin
@US-git.commits.create @UC-git.commits.manage-commit-history @P1 @versioning @commit
Feature: US-git.commits.create Фиксация изменений с сообщением коммита

  Background:
    Given пользователь аутентифицирован как "Инженер знаний"
    And открыта онтология "TestOntology"

  Scenario: Инженер знаний создаёт коммит с сообщением
    Given пользователь создал класс "Vehicle"
    When пользователь открывает панель фиксации изменений
    And вводит сообщение коммита "Add Vehicle class"
    And подтверждает создание коммита
    Then система создаёт коммит с уникальным хэшем
    And в истории коммитов появляется запись "Add Vehicle class"
    And текущая версия онтологии обновляется
```
