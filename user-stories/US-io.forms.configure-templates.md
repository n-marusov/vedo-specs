<a id="us-io.forms.configure-templates"></a>
# US-io.forms.configure-templates: Настройка шаблонов форм импорта

```gherkin
@US-io.forms.configure-templates @UC-io.forms.configure-import-form-templates @P2 @io @forms
Feature: US-io.forms.configure-templates Настройка шаблонов форм импорта

  Background:
    Given пользователь аутентифицирован как "ИТ-архитектор"

  Scenario: Пользователь создает шаблон импорта для класса
    When пользователь открывает конструктор форм импорта
    And добавляет поля шаблона и маппинг на свойства класса "PowerPlant"
    And сохраняет шаблон "PowerPlantImportV1"
    Then система сохраняет шаблон
    And шаблон доступен при следующем импорте
```
