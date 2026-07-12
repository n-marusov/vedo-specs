<a id="us-browse.public.view-accessible"></a>
# US-browse.public.view-accessible: Просмотр опубликованной онтологии без авторизации (accessible)

```gherkin
@US-browse.public.view-accessible @A10 @P0 @accessibility @publish
Feature: US-browse.public.view-accessible Просмотр опубликованной онтологии без авторизации (accessible)

  Background:
    Given пользователь использует screen reader
    And существует опубликованная онтология "ProductOntology" с публичной ссылкой

  Scenario: Внешний пользователь с ограничениями просматривает опубликованную онтологию
    When пользователь открывает публичную ссылку
    Then система не запрашивает авторизацию
    And дерево классов доступно через screen reader
    And полнотекстовый поиск работает с клавиатуры
    And навигация по дереву и графу соответствует требованиям доступности
```
