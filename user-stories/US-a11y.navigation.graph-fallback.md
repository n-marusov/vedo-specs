<a id="us-a11y.navigation.graph-fallback"></a>
# US-a11y.navigation.graph-fallback: Навигация по графу через screen reader (fallback)

```gherkin
@US-a11y.navigation.graph-fallback @A2 @P0 @accessibility @navigation
Feature: US-a11y.navigation.graph-fallback Навигация по графу через screen reader (fallback)

  Background:
    Given пользователь использует screen reader
    And открыта онтология "TestOntology"

  Scenario: Пользователь screen reader переключает классы на графе
    When граф отображается
    Then screen reader корректно читает элементы графа
    And пользователь может переключаться между классами на графе
```
