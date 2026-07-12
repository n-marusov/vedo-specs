<a id="us-a11y.comments.view-add"></a>
# US-a11y.comments.view-add: Просмотр и добавление комментариев с клавиатуры

```gherkin
@US-a11y.comments.view-add @A7 @P1 @accessibility @collaboration
Feature: US-a11y.comments.view-add Просмотр и добавление комментариев с клавиатуры

  Background:
    Given пользователь использует keyboard-only навигацию
    And открыта онтология "TestOntology"

  Scenario: Незрячий рецензент читает и оставляет комментарии
    Given класс "Person" содержит комментарии от нескольких авторов
    When пользователь открывает панель комментариев класса
    Then screen reader объявляет автора и текст каждого комментария в хронологическом порядке
    When пользователь вводит новый комментарий
    And отправляет его через Enter
    Then комментарий сохранён
    And screen reader объявляет подтверждение
```
