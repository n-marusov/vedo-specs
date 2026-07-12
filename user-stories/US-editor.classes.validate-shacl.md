<a id="us-editor.classes.validate-shacl"></a>
# US-editor.classes.validate-shacl: Проверка класса на соответствие SHACL-правилам

```gherkin
@US-editor.classes.validate-shacl @UC-editor.classes.validate-ontology-with-shacl @P1 @quality @validation @shacl
Feature: US-editor.classes.validate-shacl Проверка класса на соответствие SHACL-правилам

  Background:
    Given пользователь аутентифицирован как "Инженер знаний"
    And открыта онтология "TestOntology"

  Scenario: Инженер знаний запускает SHACL-валидацию класса
    Given существует SHACL-правило "У Person должен быть заполнен ИНН"
    And класс "Person" содержит 10 индивидов
    And 3 индивида нарушают SHACL-правило
    When пользователь выбирает класс "Person"
    And запускает проверку корректности данных
    Then система отображает отчёт "Нарушений: 3"
    And система показывает список нарушителей с указанием правил
```
