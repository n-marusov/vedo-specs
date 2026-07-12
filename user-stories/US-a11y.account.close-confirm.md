<a id="us-a11y.account.close-confirm"></a>
# US-a11y.account.close-confirm: Закрытие аккаунта через typed confirmation с клавиатуры

```gherkin
@US-a11y.account.close-confirm @A11 @P1 @accessibility @security
Feature: US-a11y.account.close-confirm Закрытие аккаунта через typed confirmation с клавиатуры

  Background:
    Given пользователь использует keyboard-only навигацию

  Scenario: Пользователь с моторными нарушениями закрывает аккаунт
    When пользователь открывает настройки аккаунта
    And активирует процедуру закрытия аккаунта
    Then система запрашивает typed confirmation
    When пользователь вводит подтверждение с клавиатуры
    And подтверждает действие
    Then система выполняет закрытие аккаунта
    And screen reader объявляет статус операции
```
