<a id="us-editor.classes.hierarchy-drag-drop"></a>
# US-editor.classes.hierarchy-drag-drop: Изменение иерархии классов через drag-n-drop

```gherkin
@US-editor.classes.hierarchy-drag-drop @UC-editor.classes.manage-class-lifecycle @P1 @ontology @class @ui
Feature: US-editor.classes.hierarchy-drag-drop Изменение иерархии классов через drag-n-drop

  Background:
    Given пользователь аутентифицирован как "Инженер знаний"
    And открыта онтология "TestOntology"

  Scenario: Инженер знаний перемещает класс в иерархии через drag-n-drop
    Given существует класс "Animal"
    And существует класс "Mammal" с родителем "Animal"
    And существует класс "Dog" с родителем "Mammal"
    When пользователь перетаскивает класс "Dog" на класс "Animal"
    And подтверждает изменение
    Then система меняет родителя "Dog" на "Animal"
    And класс "Dog" отображается под "Animal"

  Scenario: Система отклоняет циклический drag-n-drop
    Given существует класс "A" с родителем "B"
    And существует класс "B" с родителем "C"
    When пользователь перетаскивает класс "C" на класс "A"
    Then система показывает ошибку "Cycle detected"
    And иерархия не изменяется

  Scenario: Система предупреждает при потере наследуемых свойств
    Given существует класс "Mammal" с DatatypeProperty "hasFur"
    And существует класс "Dog" с родителем "Mammal"
    When пользователь перетаскивает класс "Dog" из "Mammal" в "Animal"
    Then система показывает предупреждение "Class 'Dog' will lose inherited property 'hasFur'"
    And запрашивает подтверждение
```
