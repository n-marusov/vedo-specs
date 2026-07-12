<a id="us-a11y.comments.add-keyboard"></a>
# US-a11y.comments.add-keyboard: Добавление комментария с клавиатуры

```gherkin
@US-a11y.comments.add-keyboard @E2 @P1 @accessibility @collaboration @planned
Feature: US-a11y.comments.add-keyboard Добавление комментария с клавиатуры

  Background:
    Given пользователь использует keyboard-only навигацию
    And открыта панель комментариев сущности

  Scenario: Пользователь добавляет комментарий и получает подтверждение
    When пользователь вводит текст комментария
    And отправляет комментарий через Enter
    Then система сохраняет комментарий
    And screen reader объявляет "Комментарий добавлен"
```
