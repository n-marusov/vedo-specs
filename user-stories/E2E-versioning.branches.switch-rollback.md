<a id="e2e-versioning.branches.switch-rollback"></a>
# E2E-versioning.branches.switch-rollback: Переключение веток и откат версий

```gherkin
@E2E-versioning.branches.switch-rollback @e2e @versioning @branch @P1
Feature: E2E-versioning.branches.switch-rollback Переключение веток и откат версий

  Scenario: Инженер знаний создаёт ветку, переключается, модифицирует и откатывает
    Given пользователь аутентифицирован как "Инженер знаний"
    And открыта онтология "University"
    And существует коммит "Initial ontology" на ветке "main"
    When пользователь создаёт ветку "feature/experiment" из "main"
    And переключается на ветку "feature/experiment"
    And создаёт класс "TestClass" с описанием "Experimental"
    And создаёт коммит с сообщением "Add TestClass experiment"
    Then история коммитов ветки "feature/experiment" содержит 2 записи
    When пользователь переключается на ветку "main"
    Then дерево классов не содержит "TestClass"
    When пользователь откатывает ветку "main" к начальному коммиту
    Then состояние ветки "main" соответствует начальному коммиту
    And материализованное состояние в Neo4j соответствует ожидаемому
```
