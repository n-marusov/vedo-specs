<a id="us-team.comments.feed"></a>
# US-team.comments.feed: Просмотр общей ленты комментариев проекта

```gherkin
@US-team.comments.feed @UC-team.comments.view-project-comment-feed @P1 @collaboration @comments
Feature: US-team.comments.feed Просмотр общей ленты комментариев проекта

  Background:
    Given пользователь аутентифицирован как "Maintainer"
    And в проекте существуют комментарии к классам, индивидам и Merge Request

  Scenario: Пользователь открывает ленту и применяет фильтры
    When пользователь открывает ленту комментариев проекта
    And фильтрует по сущности "Merge Request"
    And фильтрует по статусу "не разрешен"
    Then система показывает только подходящие комментарии в хронологическом порядке
    And новые комментарии появляются в ленте без перезагрузки
```
