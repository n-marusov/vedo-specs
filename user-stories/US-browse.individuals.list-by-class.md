<a id="us-browse.individuals.list-by-class"></a>
# US-browse.individuals.list-by-class: Просмотр списка индивидов, отфильтрованных по классу

```gherkin
@US-browse.individuals.list-by-class @UC-abox.individuals.manage-individual-lifecycle @P1 @ontology @individual @browse
Feature: US-browse.individuals.list-by-class Просмотр списка индивидов, отфильтрованных по классу

  Background:
    Given пользователь аутентифицирован как "Инженер знаний"
    And открыта онтология "TestOntology"

  Scenario: Инженер знаний просматривает индивидов выбранного класса
    Given существует класс "Person"
    And существует 25 индивидов класса "Person"
    When пользователь открывает список индивидов для класса "Person"
    Then система отображает первые 20 индивидов
    And пагинация показывает "1/2"
    And каждый индивид отображает свой ID и label

  Scenario: Система отображает пустой список для класса без индивидов
    Given существует класс "AbstractConcept"
    And нет индивидов класса "AbstractConcept"
    When пользователь открывает список индивидов для класса "AbstractConcept"
    Then система отображает пустой список
    And сообщение "No individuals found for class 'AbstractConcept'"

  Scenario: Инженер знаний переходит на следующую страницу
    Given существует класс "Person" с 25 индивидами
    When пользователь открывает вторую страницу списка
    Then система отображает индивиды 21-25
```
