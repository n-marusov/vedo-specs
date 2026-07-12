<a id="us-io.export.docx"></a>
# US-io.export.docx: Экспорт онтологии в DOCX

```gherkin
@US-io.export.docx @UC-io.export.export-ontology-to-docx @P2 @io @docx
Feature: US-io.export.docx Экспорт онтологии в DOCX

  Background:
    Given открыта онтология "TestOntology"

  Scenario: Пользователь формирует документ DOCX по онтологии
    When пользователь выбирает экспорт в формат DOCX
    And выбирает включение разделов классов и свойств
    Then система формирует DOCX-документ
    And документ доступен для скачивания
```
