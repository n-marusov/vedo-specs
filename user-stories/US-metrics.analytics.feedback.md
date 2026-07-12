<a id="us-metrics.analytics.feedback"></a>
# US-metrics.analytics.feedback: Анализ дашборда обратной связи

```gherkin
@US-metrics.analytics.feedback @UC-support.feedback.review-feedback-analytics @P2 @support @analytics
Feature: US-metrics.analytics.feedback Анализ дашборда обратной связи

  Background:
    Given в системе накоплены тикеты и in-app отзывы за период

  Scenario: Product Manager анализирует тренды обратной связи
    When Product Manager открывает дашборд обратной связи
    Then система показывает классификацию отзывов по категориям
    And отображает тренды удовлетворенности и топ проблем
    And данные доступны для приоритезации roadmap
```
