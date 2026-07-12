<a id="e2e-editor.workflow.full-cycle"></a>
# E2E-editor.workflow.full-cycle: Полный цикл работы инженера знаний

```gherkin
@E2E-editor.workflow.full-cycle @e2e @workflow @P0
Feature: E2E-editor.workflow.full-cycle Полный цикл работы инженера знаний

  Scenario: Создание онтологии от класса до коммита
    Given пользователь аутентифицирован как "Инженер знаний"
    And создана новая онтология "University"
    When пользователь создаёт класс "Person" с описанием "A person"
    And создаёт класс "Student" с родителем "Person"
    And создаёт класс "Professor" с родителем "Person"
    And создаёт DatatypeProperty "name" для класса "Person" с типом "string"
    And создаёт ObjectProperty "advises" с domain "Professor" и range "Student"
    And создаёт индивид "John" класса "Professor"
    And создаёт индивид "Alice" класса "Student"
    And создаёт связь "advises" от "John" к "Alice"
    Then граф содержит узлы "Professor", "Student", "Person"
    And связь "advises" соединяет "John" и "Alice"
    When пользователь создаёт коммит с сообщением "Initial university ontology"
    Then коммит создан и версия обновлена
```
