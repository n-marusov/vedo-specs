<a id="us-support.telemetry.opt-in"></a>
# US-support.telemetry.opt-in: Управление opt-in телеметрии использования

```gherkin
@US-support.telemetry.opt-in @UC-support.telemetry.manage-usage-telemetry-opt-in @P2 @support @telemetry
Feature: US-support.telemetry.opt-in Управление opt-in телеметрии использования

  Background:
    Given пользователь имеет доступ к настройкам телеметрии tenant

  Scenario: Администратор отключает сбор телеметрии
    When администратор открывает настройки телеметрии
    And переключает режим на "off"
    Then система прекращает сбор usage-метрик для tenant
    And сохраняет статус согласия
```
