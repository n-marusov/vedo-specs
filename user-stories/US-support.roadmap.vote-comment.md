<a id="us-support.roadmap.vote-comment"></a>
# US-support.roadmap.vote-comment: Голосование и комментарии в публичном roadmap

```gherkin
@US-support.roadmap.vote-comment @UC-support.roadmap.vote-and-comment-roadmap-items @P2 @support @roadmap
Feature: US-support.roadmap.vote-comment Голосование и комментарии в публичном roadmap

  Background:
    Given опубликован публичный roadmap VEDO Core

  Scenario: Пользователь голосует за feature и оставляет комментарий
    When пользователь открывает карточку feature "Advanced SPARQL templates"
    And голосует за feature
    And добавляет комментарий с обоснованием
    Then система учитывает голос
    And комментарий отображается в обсуждении roadmap
```
