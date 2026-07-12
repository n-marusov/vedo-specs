<a id="us-a11y.navigation.tree-reader"></a>
# US-a11y.navigation.tree-reader: Навигация по дереву классов через screen reader

```gherkin
@US-a11y.navigation.tree-reader @A3 @P0 @accessibility @navigation
Feature: US-a11y.navigation.tree-reader Навигация по дереву классов через screen reader

  Background:
    Given пользователь использует screen reader
    And открыта онтология "TestOntology" с вложенными классами

  Scenario: Пользователь screen reader навигируется по дереву
    When фокус находится на дереве классов
    Then screen reader объявляет текущий выбранный класс, его уровень вложенности и состояние (свёрнут/развёрнут)
    When пользователь раскрывает узел дерева
    Then screen reader объявляет дочерние классы
    And фокус перемещается на первый дочерний класс
```
