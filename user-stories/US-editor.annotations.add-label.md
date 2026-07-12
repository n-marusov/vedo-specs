<a id="us-editor.annotations.add-label"></a>
# US-editor.annotations.add-label: Добавление rdfs:label на нескольких языках

```gherkin
@US-editor.annotations.add-label @UC-editor.properties.manage-ontology-annotations @P1 @ontology @annotation @multilingual
Feature: US-editor.annotations.add-label Добавление rdfs:label на нескольких языках

  Background:
    Given пользователь аутентифицирован как "Инженер знаний"
    And открыта онтология "TestOntology"

  Scenario: Инженер знаний добавляет многоязычные label
    Given существует класс "Product"
    When пользователь открывает класс "Product" для редактирования
    And вводит rdfs:label "Товар" с языком "ru"
    And вводит rdfs:label "Product" с языком "en"
    And сохраняет изменения
    Then при выборе языка "ru" отображается "Товар"
    And при выборе языка "en" отображается "Product"
    And экспорт Turtle содержит языковые теги "@ru" и "@en"
```
