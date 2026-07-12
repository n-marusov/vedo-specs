<a id="us-editor.properties.create-object"></a>
# US-editor.properties.create-object: Создание ObjectProperty с domain и range

```gherkin
@US-editor.properties.create-object @UC-editor.properties.manage-property-lifecycle @P0 @ontology @property @object
Feature: US-editor.properties.create-object Создание ObjectProperty с domain и range

  Background:
    Given пользователь аутентифицирован как "Инженер знаний"
    And открыта онтология "TestOntology"

  Scenario: Инженер знаний создаёт ObjectProperty между классами
    Given существует класс "Person"
    And существует класс "Organization"
    When пользователь создаёт свойство типа "ObjectProperty"
    And вводит наименование "РаботаетВ"
    And устанавливает domain "Person"
    And устанавливает range "Organization"
    And сохраняет свойство
    Then свойство "РаботаетВ" создано
    And свойство отображается в списке свойств класса "Person"
    And для индивидов класса "Person" можно выбрать индивид класса "Organization"
```
