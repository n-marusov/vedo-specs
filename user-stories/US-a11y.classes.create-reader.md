<a id="us-a11y.classes.create-reader"></a>
# US-a11y.classes.create-reader: Создание класса через screen reader

```gherkin
@US-a11y.classes.create-reader @A4 @P0 @accessibility @editing
Feature: US-a11y.classes.create-reader Создание класса через screen reader

  Background:
    Given пользователь использует screen reader
    And открыта онтология "TestOntology"

  Scenario: Пользователь screen reader создаёт класс
    When пользователь активирует кнопку "Создать класс"
    Then screen reader объявляет открытие формы создания
    When пользователь заполняет поле Label
    And заполняет поле Comment
    And выбирает родительский класс из autocomplete с клавиатуры
    And сохраняет класс
    Then система создаёт класс
    And screen reader объявляет подтверждение создания
    And класс отображается в дереве под выбранным родителем
```
