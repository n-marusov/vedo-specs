<a id="us-browse.search.fulltext"></a>
# US-browse.search.fulltext: Поиск классов и свойств по наименованию

```gherkin
@US-browse.search.fulltext @UC-browse.search.search-ontology-elements @P0 @ontology @search
Feature: US-browse.search.fulltext Поиск классов и свойств по наименованию

  Background:
    Given пользователь аутентифицирован как "Аналитик"
    And открыта онтология "TestOntology"

  Scenario: Аналитик ищет элемент онтологии по части наименования
    Given онтология содержит классы "Person", "Organization", "Product"
    And онтология содержит свойство "РаботаетВ"
    When пользователь вводит "Per" в строку поиска
    Then система показывает "Person" в результатах поиска
    And система не показывает нерелевантные элементы
    When пользователь выбирает "Person"
    Then открывается карточка класса "Person"
```
