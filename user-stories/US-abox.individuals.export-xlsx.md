<a id="us-abox.individuals.export-xlsx"></a>
# US-abox.individuals.export-xlsx: Экспорт выбранных индивидов в XLSX

```gherkin
@US-abox.individuals.export-xlsx @UC-abox.individuals.export-individuals-to-xlsx @P1 @abox @export
Feature: US-abox.individuals.export-xlsx Экспорт выбранных индивидов в XLSX

  Background:
    Given пользователь аутентифицирован как "Инженер знаний"
    And открыта онтология "TestOntology"
    And в классе "PowerPlant" существуют индивиды

  Scenario: Пользователь экспортирует выбранных индивидов с настраиваемыми колонками
    When пользователь выбирает индивидов класса "PowerPlant"
    And выбирает колонки "label", "region", "capacity"
    And запускает экспорт в XLSX
    Then система формирует XLSX-файл
    And файл содержит только выбранных индивидов и колонки
```
