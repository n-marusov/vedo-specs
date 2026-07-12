<a id="us-io.xlsx.import-export"></a>
# US-io.xlsx.import-export: Импорт и экспорт онтологии в XLSX

```gherkin
@US-io.xlsx.import-export @UC-io.import.import-and-export-ontology-xlsx @P1 @io @xlsx
Feature: US-io.xlsx.import-export Импорт и экспорт онтологии в XLSX

  Background:
    Given пользователь аутентифицирован как "Инженер знаний"
    And открыта онтология "TestOntology"

  Scenario: Пользователь импортирует данные из XLSX по шаблону
    When пользователь выбирает XLSX-файл импорта
    And применяет шаблон маппинга колонок к свойствам
    Then система импортирует данные
    And возвращает отчет об успешных и проблемных строках

  Scenario: Пользователь экспортирует данные в XLSX
    When пользователь запускает экспорт данных онтологии в XLSX
    Then система формирует файл XLSX
    And файл доступен для скачивания
```
