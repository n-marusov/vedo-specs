<a id="us-a11y.classes.edit-validated"></a>
# US-a11y.classes.edit-validated: Редактирование класса с доступной валидацией

```gherkin
@US-a11y.classes.edit-validated @C2 @P1 @accessibility @editing @planned
Feature: US-a11y.classes.edit-validated Редактирование класса с доступной валидацией

  Background:
    Given пользователь использует screen reader
    And существует класс "Person"

  Scenario: Пользователь редактирует поля класса и сохраняет изменения
    When пользователь открывает форму редактирования класса
    And изменяет Label и Comment
    And нажимает "Сохранить"
    Then система сохраняет изменения
    And screen reader объявляет статус успешного сохранения
```
