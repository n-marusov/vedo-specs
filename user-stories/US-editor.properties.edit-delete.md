<a id="us-editor.properties.edit-delete"></a>
# US-editor.properties.edit-delete: Редактирование и удаление ObjectProperty и DatatypeProperty

```gherkin
@US-editor.properties.edit-delete @UC-editor.properties.manage-property-lifecycle @P0 @ontology @property @edit
Feature: US-editor.properties.edit-delete Редактирование и удаление ObjectProperty и DatatypeProperty

  Background:
    Given пользователь аутентифицирован как "Инженер знаний"
    And открыта онтология "TestOntology"

  Scenario: Инженер знаний редактирует ObjectProperty
    Given существует ObjectProperty "hasChild" с domain "Person" и range "Person"
    When пользователь изменяет domain "hasChild" на "Animal"
    And изменяет характеристику на "transitive"
    And сохраняет изменения
    Then система обновляет ObjectProperty "hasChild"
    And domain "hasChild" теперь "Animal"
    And характеристика "transitive" активна

  Scenario: Инженер знаний редактирует DatatypeProperty
    Given существует DatatypeProperty "age" с domain "Person" и xsd:type "integer"
    When пользователь изменяет xsd:type "age" на "string"
    And сохраняет изменения
    Then система обновляет DatatypeProperty "age"
    And xsd:type "age" теперь "string"

  Scenario: Инженер знаний удаляет свойство
    Given существует ObjectProperty "obsoleteRelation"
    When пользователь удаляет свойство "obsoleteRelation"
    Then система удаляет "obsoleteRelation"
    And свойство не отображается в списке свойств

  Scenario: Система предупреждает при удалении используемого свойства
    Given существует ObjectProperty "hasParent"
    And свойство "hasParent" используется классами "Person" и "Animal"
    When пользователь пытается удалить "hasParent"
    Then система показывает предупреждение "Property 'hasParent' is in use by 2 classes"
    And запрашивает подтверждение
```
