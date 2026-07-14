<a id="e2e-browse.graph-view"></a>
# E2E-browse.graph-view: Визуализация графа онтологии

```gherkin
@E2E-browse.graph-view @e2e @browse @graph @P1
Feature: E2E-browse.graph-view Визуализация графа онтологии

  Scenario: Просмотр и навигация по графу онтологии
    Given пользователь аутентифицирован как "Инженер знаний"
    And открыта онтология "University" на ветке "main"
    And онтология содержит классы "Person", "Student", "Professor", "Organization"
    And онтология содержит ObjectProperty "advises" с domain "Professor" и range "Student"
    When пользователь открывает вкладку "Graph" в рабочей области
    Then отображается граф с узлами "Person", "Student", "Professor", "Organization"
    And отображается ребро "advises" между "Professor" и "Student"
    When пользователь увеличивает масштаб графа (zoom in)
    Then узлы графа становятся крупнее
    And метки узлов остаются читаемыми
    When пользователь кликает на узел "Person"
    Then отображается панель деталей узла "Person"
    And панель содержит информацию о классе: имя, родители, свойства
    When пользователь уменьшает масштаб графа (zoom out)
    Then граф возвращается к исходному масштабу
    When пользователь перетаскивает узел "Student"
    Then позиция узла "Student" изменяется
```
