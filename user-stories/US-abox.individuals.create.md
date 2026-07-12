<a id="us-abox.individuals.create"></a>
# US-abox.individuals.create: Создание индивида класса с заполнением свойств

```gherkin
@US-abox.individuals.create @UC-abox.individuals.manage-individual-lifecycle @P1 @ontology @individual @create
Feature: US-abox.individuals.create Создание индивида класса с заполнением свойств

  Background:
    Given пользователь аутентифицирован как "Инженер знаний"
    And открыта онтология "TestOntology"

  Scenario: Инженер знаний создаёт индивид и заполняет свойства
    Given существует класс "Person" со свойством "ФИО"
    And существует класс "Organization"
    And существует индивид "ООО Альфа" класса "Organization"
    When пользователь создаёт индивид класса "Person"
    And вводит "ФИО" = "Иванов Иван Иванович"
    And выбирает "РаботаетВ" = "ООО Альфа"
    And сохраняет индивид
    Then система создаёт индивид с уникальным ID
    And индивид отображается в списке объектов класса "Person"

  Scenario: Система отклоняет индивид без обязательного свойства
    Given класс "Person" имеет обязательное свойство "ФИО"
    When пользователь создаёт индивид класса "Person" без значения "ФИО"
    Then система показывает ошибку "Property 'ФИО' is required"
    And индивид не создаётся
```
