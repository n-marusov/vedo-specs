<a id="us-a11y.classes.property-panel"></a>
# US-a11y.classes.property-panel: Просмотр панели свойств класса с клавиатуры

```gherkin
@US-a11y.classes.property-panel @B3 @P1 @accessibility @navigation @planned
Feature: US-a11y.classes.property-panel Просмотр панели свойств класса с клавиатуры

  Background:
    Given пользователь использует keyboard-only навигацию
    And открыт экран карточки класса

  Scenario: Пользователь переключает вкладки панели свойств
    When пользователь переходит фокусом в панель свойств
    And переключает вкладки стрелками в tablist
    Then система показывает содержимое выбранной вкладки
    And screen reader объявляет активную вкладку и список свойств
```
