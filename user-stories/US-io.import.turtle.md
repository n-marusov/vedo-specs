<a id="us-io.import.turtle"></a>
# US-io.import.turtle: Импорт онтологии из Turtle-файла

```gherkin
@US-io.import.turtle @UC-io.import.import-and-export-ontology-data @P0 @exchange @import @turtle
Feature: US-io.import.turtle Импорт онтологии из Turtle-файла

  Background:
    Given пользователь аутентифицирован как "Инженер знаний"
    And открыта онтология "TestOntology"

  Scenario: Инженер знаний импортирует валидный Turtle-файл
    Given пользователь имеет файл "ontology.ttl" с корректной онтологией
    When пользователь выбирает импорт из Turtle
    And загружает файл "ontology.ttl"
    And выбирает стратегию "Слияние"
    And запускает импорт
    Then система отображает прогресс импорта
    And после завершения показывает сообщение "Импорт успешно завершён"
    And импортированные классы отображаются в дереве классов

  Scenario: Система отклоняет Turtle-файл с синтаксической ошибкой
    Given пользователь имеет файл "invalid.ttl" с синтаксической ошибкой
    When пользователь загружает файл "invalid.ttl"
    Then система показывает ошибку парсинга Turtle с номером строки
    And онтология остаётся неизменной
```
