<a id="us-a11y.publish.view-public"></a>
# US-a11y.publish.view-public: Открытие публичной ссылки на онтологию

```gherkin
@US-a11y.publish.view-public @F1 @P1 @accessibility @publish @planned
Feature: US-a11y.publish.view-public Открытие публичной ссылки на онтологию

  Background:
    Given существует опубликованная онтология с публичным URL

  Scenario: Пользователь открывает публичную онтологию без авторизации
    When пользователь переходит по публичной ссылке
    Then система открывает страницу онтологии без логина
    And страница содержит доступную навигацию и заголовок уровня H1
```
