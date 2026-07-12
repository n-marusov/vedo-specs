<a id="us-admin.endpoints.create-bind"></a>
# US-admin.endpoints.create-bind: Создание точки доступа и привязка хранилища

```gherkin
@US-admin.endpoints.create-bind @UC-admin.system.manage-platform-configuration @P1 @administration @endpoint @storage
Feature: US-admin.endpoints.create-bind Создание точки доступа и привязка хранилища

  Background:
    Given пользователь аутентифицирован как "Администратор"

  Scenario: Администратор создаёт точку доступа и привязывает хранилище
    Given существует хранилище "PostgreSQL Store" типа "postgresql_jsonb"
    When администратор создаёт точку доступа "Production"
    And привязывает хранилище "PostgreSQL Store" к точке доступа "Production"
    Then точка доступа "Production" создана
    And хранилище привязано к точке доступа "Production"
    And объекты точки доступа "Production" сохраняются в PostgreSQL
```
