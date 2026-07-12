<a id="us-browse.public.view"></a>
# US-browse.public.view: Просмотр опубликованной онтологии без авторизации

```gherkin
@US-browse.public.view @UC-browse.public.view-published-ontology @P1 @publish @browse
Feature: US-browse.public.view Просмотр опубликованной онтологии без авторизации

  Background:
    Given существует опубликованная онтология "ProductOntology" с публичной ссылкой

  Scenario: Внешний пользователь просматривает опубликованную онтологию
    When пользователь открывает публичную ссылку
    Then система не запрашивает авторизацию
    And дерево классов доступно для просмотра
    And полнотекстовый поиск работает
```
