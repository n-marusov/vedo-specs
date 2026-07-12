<a id="us-editor.properties.create-datatype"></a>
# US-editor.properties.create-datatype: Создание DatatypeProperty с XSD-типом

```gherkin
@US-editor.properties.create-datatype @UC-editor.properties.manage-property-lifecycle @P0 @ontology @property @datatype
Feature: US-editor.properties.create-datatype Создание DatatypeProperty с XSD-типом

  Background:
    Given пользователь аутентифицирован как "Инженер знаний"
    And открыта онтология "TestOntology"

  Scenario: Инженер знаний создаёт DatatypeProperty для класса
    Given существует класс "Person"
    When пользователь создаёт свойство типа "DatatypeProperty"
    And вводит наименование "ФИО"
    And устанавливает range "xsd:string"
    And устанавливает minCardinality "1"
    And сохраняет свойство
    Then свойство "ФИО" создано как обязательное
    And при создании индивида класса "Person" поле "ФИО" обязательно для заполнения
```
