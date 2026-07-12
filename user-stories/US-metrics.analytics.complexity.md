<a id="us-metrics.analytics.complexity"></a>
# US-metrics.analytics.complexity: Просмотр графиков сложности онтологии

```gherkin
@US-metrics.analytics.complexity @UC-metrics.analytics.view-ontology-complexity-trends @P2 @metrics @analytics
Feature: US-metrics.analytics.complexity Просмотр графиков сложности онтологии

  Background:
    Given в системе собрана история метрик онтологии "EnterpriseOntology"

  Scenario: Пользователь анализирует тренды сложности
    When пользователь открывает раздел аналитики сложности
    And выбирает период "последние 30 дней"
    Then система отображает графики глубины, ветвления и количества orphan-классов
    And пользователь видит изменение метрик по времени
```
