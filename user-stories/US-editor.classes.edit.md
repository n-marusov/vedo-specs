<a id="us-editor.classes.edit"></a>
# US-editor.classes.edit: Редактирование свойств класса

```gherkin
@US-editor.classes.edit @UC-editor.classes.manage-class-lifecycle @P0 @ontology @class @edit
Feature: US-editor.classes.edit Редактирование свойств класса

  Background:
    Given пользователь аутентифицирован как "Инженер знаний"
    And открыта онтология "TestOntology"

  Scenario: Инженер знаний изменяет атрибуты и связи класса
    Given существует класс "Cat" с родителем "Animal"
    And существует класс "WildAnimal"
    When пользователь открывает класс "Cat" для редактирования
    And меняет родительский класс на "WildAnimal"
    And сохраняет изменения
    Then класс "Cat" отображается как подкласс "WildAnimal"
    And класс "Cat" больше не отображается как подкласс "Animal"
    And изменение попадает в diff текущей версии

  Scenario: Система отклоняет сохранение при невалидной иерархии
    Given существует класс "Animal"
    And класс "Cat" является подклассом "Animal"
    When пользователь пытается сделать "Animal" подклассом "Cat"
    And сохраняет изменения
    Then система показывает ошибку "Cycle detected"
    And изменения не сохраняются
```
