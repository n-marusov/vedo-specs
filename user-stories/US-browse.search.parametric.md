<a id="us-browse.search.parametric"></a>
# US-browse.search.parametric: Параметрический поиск индивидов по значениям свойств

```gherkin
@US-browse.search.parametric @UC-browse.search.search-ontology-elements @P0 @ontology @search @parametric
Feature: US-browse.search.parametric Параметрический поиск индивидов по значениям свойств

  Background:
    Given пользователь аутентифицирован как "Инженер знаний"
    And открыта онтология "TestOntology"

  Scenario: Инженер знаний фильтрует индивиды по значению свойства
    Given существуют индивиды класса "Person"
      | ФИО          | Город           |
      | Иванов Иван  | Москва          |
      | Петров Пётр  | Санкт-Петербург |
      | Сидоров Сидор | Москва         |
    When пользователь выбирает класс "Person"
    And выбирает фильтр по свойству "Город"
    And вводит значение "Москва"
    And запускает поиск
    Then таблица результатов содержит 2 записи
    And результаты содержат "Иванов Иван" и "Сидоров Сидор"
    And результаты не содержат "Петров Пётр"
```
