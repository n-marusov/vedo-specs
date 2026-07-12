<a id="us-admin.access.assign-role"></a>
# US-admin.access.assign-role: Owner назначает пользователю роль Editor

```gherkin
@US-admin.access.assign-role @UC-admin.access.manage-membership-and-permissions @P0 @owner @membership @rbac
Feature: US-admin.access.assign-role Owner назначает пользователю роль Editor

  Background:
    Given пользователь аутентифицирован как "Owner"
    And существует онтология "CustomerOntology"

  Scenario: Owner назначает пользователю роль Editor
    Given пользователь "Bob" зарегистрирован в Keycloak
    When Owner открывает управление участниками онтологии "CustomerOntology"
    And добавляет пользователя "Bob"
    And назначает роль "Editor"
    And сохраняет изменения
    Then пользователь "Bob" может редактировать классы в "CustomerOntology"
    And пользователь "Bob" не получает прав Owner

  Scenario: Система запрещает управление членством без роли Owner
    Given пользователь "Carol" имеет роль "Maintainer" в онтологии "CustomerOntology"
    When пользователь "Carol" пытается добавить пользователя "Bob" в онтологию "CustomerOntology"
    Then API возвращает HTTP 403 Forbidden
    And тело ответа содержит "Owner role required for membership management"

  Scenario: Система запрещает действие без роли Editor
    Given пользователь "Bob" имеет роль "Viewer" в онтологии
    When пользователь "Bob" отправляет DELETE-запрос на "/api/v1/classes/Person"
    Then API возвращает HTTP 403 Forbidden
    And тело ответа содержит "Insufficient permissions: Editor role required"
```
