<a id="us-io.reports.generate"></a>
# US-io.reports.generate: Формирование отчетов по онтологии

```gherkin
@US-io.reports.generate @UC-io.reports.generate-ontology-reports @P2 @io @reports
Feature: US-io.reports.generate Формирование отчетов по онтологии

  Background:
    Given в системе есть данные по онтологии "EnterpriseOntology"

  Scenario: Пользователь формирует отчет по выбранному периоду
    When пользователь выбирает тип отчета "Ontology Summary"
    And указывает период "последний месяц"
    Then система формирует отчет
    And отчет доступен для просмотра и скачивания
```
