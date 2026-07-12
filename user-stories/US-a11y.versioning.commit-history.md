<a id="us-a11y.versioning.commit-history"></a>
# US-a11y.versioning.commit-history: Просмотр истории коммитов и diff с клавиатуры

```gherkin
@US-a11y.versioning.commit-history @A8 @P1 @accessibility @versioning
Feature: US-a11y.versioning.commit-history Просмотр истории коммитов и diff с клавиатуры

  Background:
    Given пользователь использует screen reader
    And открыта онтология "TestOntology" с историей коммитов

  Scenario: Пользователь screen reader просматривает diff
    When пользователь открывает историю коммитов
    And выбирает два коммита для сравнения
    Then diff отображается в текстовом представлении
    And screen reader объявляет добавленные, удалённые и изменённые сущности
```
