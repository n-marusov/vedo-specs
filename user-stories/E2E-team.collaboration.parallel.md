<a id="e2e-team.collaboration.parallel"></a>
# E2E-team.collaboration.parallel: Командная работа над онтологией

```gherkin
@E2E-team.collaboration.parallel @e2e @P1
Feature: E2E-team.collaboration.parallel Командная работа над онтологией

  Scenario: Параллельная работа через ветки, MR и комментарии
    Given пользователь "Alice" открыла онтологию "Project"
    And пользователь "Bob" открыл ту же онтологию
    When пользователь "Alice" создаёт ветку "feature/task-fields"
    And пользователь "Bob" создаёт ветку "feature/task-status"
    And пользователь "Alice" редактирует класс "Task" в своей ветке
    And пользователь "Bob" редактирует класс "Task" в своей ветке
    And пользователь "Alice" создаёт Merge Request в "main"
    Then пользователь "Bob" видит Merge Request и семантический diff
    When пользователь "Bob" добавляет комментарий "@Alice check status consistency"
    Then пользователь "Alice" получает уведомление
    When пользователь "Alice" обновляет изменения и повторно отправляет на ревью
    Then после одобрения Merge Request изменения сливаются в "main"
```
