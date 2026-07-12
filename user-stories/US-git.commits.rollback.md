<a id="us-git.commits.rollback"></a>
# US-git.commits.rollback: Откат онтологии к предыдущему коммиту

```gherkin
@US-git.commits.rollback @UC-git.commits.manage-commit-history @P1 @versioning @rollback
Feature: US-git.commits.rollback Откат онтологии к предыдущему коммиту

  Background:
    Given пользователь аутентифицирован как "Инженер знаний"
    And открыта онтология "TestOntology"

  Scenario: Инженер знаний откатывает онтологию к предыдущему коммиту
    Given история коммитов содержит коммит "abc123" с классом "Vehicle"
    And история коммитов содержит предыдущий коммит "def456" без класса "Vehicle"
    When пользователь выбирает коммит "def456"
    And подтверждает откат
    Then система показывает diff отката
    And создаёт новый коммит "Rollback to def456"
    And класс "Vehicle" отсутствует в текущем состоянии онтологии
```
