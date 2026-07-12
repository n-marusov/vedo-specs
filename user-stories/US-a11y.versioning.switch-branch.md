<a id="us-a11y.versioning.switch-branch"></a>
# US-a11y.versioning.switch-branch: Переключение веток с клавиатуры

```gherkin
@US-a11y.versioning.switch-branch @D3 @P1 @accessibility @versioning @planned
Feature: US-a11y.versioning.switch-branch Переключение веток с клавиатуры

  Background:
    Given пользователь находится в разделе версионирования

  Scenario: Пользователь выбирает другую ветку через селектор
    When пользователь открывает селектор веток
    And выбирает целевую ветку стрелками и Enter
    Then система переключает контекст на выбранную ветку
    And screen reader объявляет имя активной ветки
```
