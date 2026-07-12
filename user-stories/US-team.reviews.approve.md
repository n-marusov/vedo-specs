<a id="us-team.reviews.approve"></a>
# US-team.reviews.approve: Ревью Merge Request перед слиянием

```gherkin
@US-team.reviews.approve @UC-team.reviews.review-merge-request @P1 @review
Feature: US-team.reviews.approve Ревью Merge Request перед слиянием

  Background:
    Given пользователи "Alice" и "Bob" аутентифицированы как "Инженер знаний"
    And открыта онтология "Project"

  Scenario: Ревьюер одобряет Merge Request
    Given пользователь "Alice" создала ветку "feature/person-updates"
    And пользователь "Alice" изменила класс "Person" в своей ветке
    And пользователь "Alice" создала Merge Request в ветку "main"
    When пользователь "Bob" открывает Merge Request
    And просматривает семантический diff
    And нажимает "Approve"
    Then статус Merge Request становится "Approved"
    And Merge Request готов к слиянию

  Scenario: Система помечает Merge Request как конфликтный
    Given пользователь "Alice" создала Merge Request из ветки "feature/person-updates" в "main"
    And в ветке "main" есть несовместимые изменения того же класса "Person"
    When пользователь "Alice" запускает слияние Merge Request
    Then система показывает сообщение "Merge conflict detected"
    And Merge Request остаётся открытым до разрешения конфликта
```
