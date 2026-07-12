<a id="us-a11y.search.fulltext-keyboard"></a>
# US-a11y.search.fulltext-keyboard: Полнотекстовый поиск с клавиатуры

```gherkin
@US-a11y.search.fulltext-keyboard @A3 @P0 @accessibility @search
Feature: US-a11y.search.fulltext-keyboard Полнотекстовый поиск с клавиатуры

  Background:
    Given пользователь использует keyboard-only навигацию
    And открыта онтология "TestOntology"

  Scenario: Пользователь выполняет поиск с клавиатуры
    When пользователь фокусируется на строке поиска через Tab
    And вводит поисковый запрос с клавиатуры
    Then система отображает автодополнение
    And screen reader объявляет количество результатов
    When пользователь навигирует по результатам стрелками
    And открывает карточку сущности через Enter
    Then система открывает карточку выбранной сущности
```
