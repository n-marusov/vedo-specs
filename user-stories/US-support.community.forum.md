<a id="us-support.community.forum"></a>
# US-support.community.forum: Участие в community-форуме

```gherkin
@US-support.community.forum @UC-support.community.participate-in-community-forum @P2 @support @community
Feature: US-support.community.forum Участие в community-форуме

  Background:
    Given пользователь имеет доступ к community-форуму

  Scenario: Пользователь создает тему и получает ответ
    When пользователь создает тему с вопросом по моделированию онтологии
    Then тема появляется в форуме
    When другой пользователь или инженер поддержки публикует ответ
    Then ответ отображается в той же теме
```
