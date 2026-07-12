<a id="us-support.feedback.in-app-nps"></a>
# US-support.feedback.in-app-nps: Отправка in-app feedback и NPS

```gherkin
@US-support.feedback.in-app-nps @UC-support.feedback.submit-in-app-feedback-and-nps @P1 @support @feedback
Feature: US-support.feedback.in-app-nps Отправка in-app feedback и NPS

  Background:
    Given пользователь работает в интерфейсе VEDO Core

  Scenario: Пользователь отправляет отзыв через in-app виджет
    When пользователь открывает виджет "Оставить отзыв"
    And вводит текст обратной связи
    And отправляет форму
    Then система сохраняет отзыв
    And прикладывает контекст URL, действия и trace_id

  Scenario: Пользователь отправляет оценку NPS
    When система показывает форму NPS
    And пользователь выбирает оценку и комментарий
    Then система сохраняет NPS-ответ
```
