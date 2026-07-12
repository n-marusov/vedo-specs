<a id="us-team.comments.visibility"></a>
# US-team.comments.visibility: Ограничение видимости комментариев по правам доступа

```gherkin
@US-team.comments.visibility @UC-team.comments.enforce-comment-visibility-by-access @P0 @collaboration @rbac
Feature: US-team.comments.visibility Ограничение видимости комментариев по правам доступа

  Background:
    Given комментарии привязаны к сущности "Class:SensitiveAsset"
    And пользователь "viewer-user" не имеет прав чтения этой сущности

  Scenario: Система скрывает комментарии при недостатке прав
    When пользователь "viewer-user" запрашивает комментарии к "Class:SensitiveAsset"
    Then система возвращает отказ доступа
    And комментарии и действия над ними не отображаются пользователю
```
