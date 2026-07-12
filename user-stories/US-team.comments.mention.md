<a id="us-team.comments.mention"></a>
# US-team.comments.mention: Добавление комментария к Merge Request с упоминанием коллеги

```gherkin
@US-team.comments.mention @UC-team.comments.discuss-merge-request-changes @P1 @comment
Feature: US-team.comments.mention Добавление комментария к Merge Request с упоминанием коллеги

  Background:
    Given пользователи "Alice" и "Bob" аутентифицированы как "Инженер знаний"
    And открыта онтология "Project"

  Scenario: Рецензент добавляет комментарий с упоминанием
    Given пользователь "Alice" создала Merge Request из ветки "feature/person-updates" в "main"
    When пользователь "Bob" открывает Merge Request
    And вводит комментарий "@Alice нужно добавить свойство 'Дата рождения'"
    And отправляет комментарий
    Then комментарий отображается у Merge Request
    And комментарий содержит автора "Bob" и timestamp
    And пользователь "Alice" получает уведомление об упоминании
```
