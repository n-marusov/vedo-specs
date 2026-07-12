<a id="us-a11y.individuals.edit-dynamic"></a>
# US-a11y.individuals.edit-dynamic: Редактирование индивида с динамическими полями

```gherkin
@US-a11y.individuals.edit-dynamic @A6 @P1 @accessibility @editing
Feature: US-a11y.individuals.edit-dynamic Редактирование индивида с динамическими полями

  Background:
    Given пользователь использует screen reader
    And открыта онтология "TestOntology" с классом, имеющим множественные свойства

  Scenario: Пользователь screen reader редактирует индивида
    When пользователь открывает карточку индивида
    And добавляет значение свойства
    Then screen reader объявляет добавление нового поля
    When пользователь заполняет несколько значений одного свойства
    Then каждое значение доступно для редактирования с клавиатуры
```
