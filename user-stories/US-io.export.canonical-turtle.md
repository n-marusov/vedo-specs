<a id="us-io.export.canonical-turtle"></a>
# US-io.export.canonical-turtle: Экспорт онтологии в канонический Turtle

```gherkin
@US-io.export.canonical-turtle @UC-io.import.import-and-export-ontology-data @P0 @exchange @export @turtle
Feature: US-io.export.canonical-turtle Экспорт онтологии в канонический Turtle

  Background:
    Given пользователь аутентифицирован как "Инженер знаний"
    And открыта онтология "TestOntology"

  Scenario: Инженер знаний экспортирует онтологию в канонический Turtle
    Given онтология содержит классы "Person" и "Organization"
    When пользователь выбирает экспорт в формат "Turtle"
    And включает каноническую сериализацию
    And запускает экспорт
    Then система генерирует файл "ontology.ttl"
    And файл содержит отсортированные триплеты
    And повторный экспорт без изменений создаёт идентичный файл
```
