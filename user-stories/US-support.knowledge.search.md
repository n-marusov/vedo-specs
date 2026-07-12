<a id="us-support.knowledge.search"></a>
# US-support.knowledge.search: Поиск решения в базе знаний и FAQ

```gherkin
@US-support.knowledge.search @UC-support.knowledge.search-knowledge-base-and-faq @P1 @support @knowledge
Feature: US-support.knowledge.search Поиск решения в базе знаний и FAQ

  Background:
    Given база знаний и FAQ опубликованы

  Scenario: Пользователь находит инструкцию без обращения в поддержку
    When пользователь вводит поисковый запрос "как восстановить backup"
    Then система показывает релевантные статьи
    And пользователь открывает инструкцию и выполняет шаги
```
