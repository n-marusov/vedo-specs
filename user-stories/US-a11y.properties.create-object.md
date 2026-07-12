<a id="us-a11y.properties.create-object"></a>
# US-a11y.properties.create-object: Создание ObjectProperty с клавиатуры

```gherkin
@US-a11y.properties.create-object @A5 @P1 @accessibility @editing
Feature: US-a11y.properties.create-object Создание ObjectProperty с клавиатуры

  Background:
    Given пользователь использует keyboard-only навигацию
    And открыта онтология "TestOntology"

  Scenario: Пользователь создаёт ObjectProperty без мыши
    When пользователь открывает форму создания свойства
    And выбирает тип "ObjectProperty"
    And заполняет Domain и Range через клавиатурный autocomplete
    And устанавливает характеристики свойства
    And сохраняет свойство
    Then система валидирует заполненные поля
    And сообщения об ошибках доступны для screen reader
    And свойство создано при успешной валидации
```
