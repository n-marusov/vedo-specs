<a id="us-abox.individuals.edit-property-values"></a>
# US-abox.individuals.edit-property-values: Установка и изменение значений свойств индивида

```gherkin
@US-abox.individuals.edit-property-values @UC-abox.individuals.manage-individual-lifecycle @P1 @ontology @individual @edit
Feature: US-abox.individuals.edit-property-values Установка и изменение значений свойств индивида

  Background:
    Given пользователь аутентифицирован как "Инженер знаний"
    And открыта онтология "TestOntology"

  Scenario: Инженер знаний устанавливает значение DatatypeProperty
    Given существует класс "Person" с DatatypeProperty "fullName"
    And существует индивид "person-001" класса "Person"
    When пользователь открывает карточку индивида "person-001"
    And устанавливает "fullName" = "Иванов Иван"
    And сохраняет изменения
    Then система сохраняет значение "fullName" = "Иванов Иван" для "person-001"
    And значение отображается в карточке индивида

  Scenario: Инженер знаний изменяет значение ReferenceProperty
    Given существует класс "Person" с ObjectProperty "worksFor"
    And существует индивид "person-001" класса "Person"
    And существует индивид "org-001" класса "Organization"
    When пользователь изменяет "worksFor" на "org-001"
    Then система обновляет ссылку "worksFor" на "org-001"
    And связь отображается в графе

  Scenario: Инженер знаний удаляет значение свойства
    Given существует индивид "person-001" со значением "email" = "test@example.com"
    When пользователь удаляет значение свойства "email"
    Then система удаляет значение "email" для "person-001"
    And свойство "email" отображается как пустое

  Scenario: Система проверяет xsd-тип при установке значения
    Given существует DatatypeProperty "age" с xsd:type "integer"
    When пользователь устанавливает "age" = "not-a-number"
    Then система показывает ошибку "Value must be of type integer"
    And значение не сохраняется
```
