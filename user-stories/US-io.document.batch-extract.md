<a id="us-io.document.batch-extract"></a>
# US-io.document.batch-extract: Пакетное извлечение из нескольких документов

```gherkin
@US-io.document.batch-extract @UC-io.import.batch-extract-ontology @P2 @io @document @batch
Feature: US-io.document.batch-extract Пакетное извлечение онтологии

  Background:
    Given пользователь аутентифицирован как "Инженер знаний"
    And открыта онтология "TestOntology"

  Scenario: Загрузка нескольких файлов разных форматов
    When пользователь выбирает 3 файла: "spec.md", "data.json", "glossary.xlsx"
    Then каждый файл обрабатывается независимо
    And результаты объединяются в один preview
    And дубликаты автоматически дедуплицируются
    And в preview отображается источник каждого шага (какой файл)

  Scenario: Ошибка в одном файле при пакетной загрузке
    When пользователь загружает файлы: "good.md" и "bad.pdf" (запаролен)
    Then "good.md" успешно обрабатывается
    And "bad.pdf" помечается ошибкой: "PDF защищён паролем"
    And preview содержит только шаги из успешного файла
    And уведомление: "1 из 2 файлов обработан успешно"

  Scenario: Конфликт имён между файлами
    When пользователь загружает файлы, содержащие класс "Customer" с разными label
    Then preview показывает конфликт: "Customer: 'Клиент' (spec.md) vs 'Покупатель' (glossary.xlsx)"
    And пользователь выбирает, какой label сохранить, или вводит свой
```
