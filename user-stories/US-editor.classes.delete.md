<a id="us-editor.classes.delete"></a>
# US-editor.classes.delete: Удаление класса с проверкой зависимостей

```gherkin
@US-editor.classes.delete @UC-editor.classes.manage-class-lifecycle @P0 @ontology @class @delete
Feature: US-editor.classes.delete Удаление класса с проверкой зависимостей

  Background:
    Given пользователь аутентифицирован как "Инженер знаний"
    And открыта онтология "TestOntology"

  Scenario: Инженер знаний удаляет класс без зависимостей
    Given существует класс "TemporaryClass" без индивидов
    And нет свойств, ссылающихся на "TemporaryClass"
    When пользователь выбирает класс "TemporaryClass"
    And подтверждает удаление
    Then класс "TemporaryClass" удалён из дерева классов
    And удаление попадает в diff текущей версии

  Scenario: Система предупреждает о зависимостях перед удалением
    Given существует класс "Person" с индивидом "JohnDoe"
    When пользователь пытается удалить класс "Person"
    Then система показывает предупреждение "Class has 1 individual(s)"
    And удаление без явного подтверждения недоступно
```
