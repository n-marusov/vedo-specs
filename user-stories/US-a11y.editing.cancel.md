<a id="us-a11y.editing.cancel"></a>
# US-a11y.editing.cancel: Отмена редактирования без сохранения

```gherkin
@US-a11y.editing.cancel @C6 @P2 @accessibility @editing @planned
Feature: US-a11y.editing.cancel Отмена редактирования без сохранения

  Background:
    Given пользователь начал редактирование сущности

  Scenario: Пользователь отменяет изменения через Cancel/Escape
    When пользователь активирует отмену редактирования
    Then система показывает подтверждение отмены
    And после подтверждения восстанавливает исходное состояние формы
```
