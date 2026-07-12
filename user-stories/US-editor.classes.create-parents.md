<a id="us-editor.classes.create-parents"></a>
# US-editor.classes.create-parents: Создание класса с несколькими родителями

```gherkin
@US-editor.classes.create-parents @UC-editor.classes.manage-class-lifecycle @P0 @ontology @class @create
Feature: US-editor.classes.create-parents Создание класса с несколькими родителями

  Background:
    Given пользователь аутентифицирован как "Инженер знаний"
    And открыта онтология "TestOntology"

  Scenario: Инженер знаний создаёт класс с несколькими родителями
    Given существует класс "Animal"
    And существует класс "Pet"
    When пользователь создаёт класс "Cat"
    And добавляет родительский класс "Animal"
    And добавляет родительский класс "Pet"
    And сохраняет класс
    Then система создаёт класс "Cat"
    And класс "Cat" отображается под родителями "Animal" и "Pet"
    And система не нарушает целостность иерархии классов

  Scenario: Система отклоняет циклическое наследование
    Given существует класс "Mammal"
    And существует класс "Dog" с родителем "Mammal"
    When пользователь пытается создать наследование, создающее цикл
    Then система показывает ошибку "Cycle detected"
    And класс не создаётся
```
