<a id="us-io.document.preview-sequence"></a>
# US-io.document.preview-sequence: Предпросмотр и редактирование sequence перед импортом

```gherkin
@US-io.document.preview-sequence @UC-io.import.extract-ontology-from-document @P1 @ui @document
Feature: US-io.document.preview-sequence Предпросмотр и редактирование sequence

  Background:
    Given пользователь загрузил файл "specs.md"
    And система сгенерировала preview sequence

  Scenario: Просмотр списка шагов
    Then пользователь видит таблицу шагов с колонками:
      | Вкл | Операция | ID        | Label    | Parent    |
      |-----|----------|-----------|----------|-----------|
      | ✓   | class    | Customer  | Клиент   | Thing     |
      | ✓   | class    | Product   | Товар    | Thing     |
      | ✓   | property | purchases | Покупает | Customer  |
    And пользователь понимает суть каждого шага без знания OWL

  Scenario: Исключение шага из импорта
    When пользователь снимает флажок с шага "create_class Product"
    Then шаг помечается как исключённый (серым цветом)
    And внизу обновляется счётчик: "Будет импортировано: 2 из 3 шагов"

  Scenario: Inline-редактирование label
    When пользователь кликает на label "Клиент"
    Then поле становится редактируемым
    When пользователь меняет на "Покупатель" и нажимает Enter
    Then label обновляется в preview

  Scenario: Предупреждение о дубликатах
    Given в онтологии уже существует класс "Customer"
    When загружен документ, содержащий класс "Customer"
    Then preview показывает предупреждение: "⚠ class Customer уже существует"
    And шаг по умолчанию исключён, но может быть включён (будет пропущен при импорте)

  Scenario: Подтверждение импорта
    When пользователь нажимает "Подтвердить импорт"
    Then система начинает процесс ApplySequence
    And отображает прогресс: "Импорт... 2/3"
    And после завершения: "✅ Импортировано: 3 сущности"
    And ссылка на коммит: "Commit #abc123"
```
