<a id="us-browse.individuals.filter-by-property"></a>
# US-browse.individuals.filter-by-property: Фильтрация списка индивидов по значениям свойств

```gherkin
@US-browse.individuals.filter-by-property @UC-abox.individuals.manage-individual-lifecycle @P1 @ontology @individual @browse
Feature: US-browse.individuals.filter-by-property Фильтрация списка индивидов по значениям свойств

  Background:
    Given пользователь аутентифицирован как "Инженер знаний"
    And открыта онтология "TestOntology"

  Scenario: Инженер знаний фильтрует индивидов по текстовому свойству
    Given существует класс "Person" с DatatypeProperty "fullName"
    And существует индивид "person-001" класса "Person" со значением "fullName" = "Иванов Иван"
    And существует индивид "person-002" класса "Person" со значением "fullName" = "Петров Пётр"
    When пользователь вводит фильтр "fullName" = "Иванов"
    Then система отображает только индивидов с "fullName" содержащим "Иванов"
    And результат содержит индивид "person-001"
    And результат не содержит индивид "person-002"

  Scenario: Инженер знаний фильтрует индивидов по числовому свойству
    Given существует класс "Product" с DatatypeProperty "price" и xsd:type "float"
    And существует 10 индивидов класса "Product" с разными ценами
    When пользователь вводит фильтр "price >= 100" и "price <= 500"
    Then система отображает индивиды с ценой от 100 до 500

  Scenario: Инженер знаний фильтрует индивидов по ссылочному свойству
    Given существует класс "Person" с ObjectProperty "worksFor"
    And существует индивид "org-001" класса "Organization"
    And существуют индивиды "Person", работающие в "org-001"
    When пользователь выбирает фильтр "worksFor" = "org-001"
    Then система отображает только индивидов, работающих в "org-001"

  Scenario: Комбинированный фильтр по нескольким свойствам
    Given существует класс "Employee"
    When пользователь применяет фильтры "department = 'IT'" и "salary > 50000" и "isActive = true"
    Then система применяет все фильтры через AND
    And отображает только подходящие индивиды

  Scenario: Система показывает пустой результат при отсутствии совпадений
    Given не существует индивидов, соответствующих фильтру
    When пользователь применяет фильтр "fullName = 'НетТакогоИмени'"
    Then система отображает пустой список
    And сообщение "No individuals match the filter criteria"
```
