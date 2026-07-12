<a id="us-editor.rules.create-visual"></a>
# US-editor.rules.create-visual: Настройка правила контроля качества данных через визуальный конструктор

```gherkin
@US-editor.rules.create-visual @UC-editor.classes.manage-data-quality-rules @P1 @quality @rule @shacl
Feature: US-editor.rules.create-visual Настройка правила контроля качества данных через визуальный конструктор

  Background:
    Given пользователь аутентифицирован как "ИТ-архитектор"
    And открыта онтология "TestOntology"

  Scenario: ИТ-архитектор создаёт правило качества данных
    Given существует класс "Person"
    When пользователь открывает визуальный конструктор правил
    And создаёт правило "ИНН должен содержать 12 цифр"
    And выбирает класс-мишень "Person"
    And добавляет условие "ИНН matches regex ^\d{12}$"
    And сохраняет правило
    Then правило сохранено и активно
    And система применяет правило при сохранении индивидов класса "Person"
```
