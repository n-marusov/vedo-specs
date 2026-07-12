<a id="us-a11y.versioning.create-commit"></a>
# US-a11y.versioning.create-commit: Создание коммита с клавиатуры

```gherkin
@US-a11y.versioning.create-commit @D2 @P1 @accessibility @versioning @planned
Feature: US-a11y.versioning.create-commit Создание коммита с клавиатуры

  Background:
    Given пользователь использует keyboard-only навигацию
    And в рабочей ветке есть изменения

  Scenario: Пользователь создает коммит с сообщением
    When пользователь открывает форму коммита
    And вводит commit message
    And подтверждает создание коммита через Enter
    Then система создает коммит
    And screen reader объявляет успешный результат
```
