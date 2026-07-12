<a id="us-git.commits.compare"></a>
# US-git.commits.compare: Сравнение двух версий онтологии

```gherkin
@US-git.commits.compare @UC-git.commits.compare-ontology-versions @P2 @versioning @diff
Feature: US-git.commits.compare Сравнение двух версий онтологии

  Background:
    Given пользователь аутентифицирован как "Инженер знаний"
    And открыта онтология "TestOntology"

  Scenario: Инженер знаний сравнивает две версии онтологии
    Given коммит "abc123" содержит класс "Person"
    And коммит "def456" содержит классы "Person" и "Organization"
    When пользователь открывает сравнение версий
    And выбирает коммит "abc123" как младший
    And выбирает коммит "def456" как старший
    And запускает сравнение
    Then система показывает добавленные, удалённые и изменённые сущности
    And класс "Organization" отображается как добавленный
```
