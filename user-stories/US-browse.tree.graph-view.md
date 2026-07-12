<a id="us-browse.tree.graph-view"></a>
# US-browse.tree.graph-view: Просмотр онтологии в виде дерева и графа

```gherkin
@US-browse.tree.graph-view @UC-browse.tree.view-ontology-tree-and-graph @P0 @ontology @visualization
Feature: US-browse.tree.graph-view Просмотр онтологии в виде дерева и графа

  Background:
    Given пользователь аутентифицирован как "Аналитик"
    And открыта онтология "TestOntology"

  Scenario: Аналитик просматривает структуру онтологии
    Given онтология содержит классы "Person", "Student", "Organization"
    And класс "Student" является подклассом "Person"
    And класс "Person" связан с "Organization" свойством "РаботаетВ"
    When пользователь открывает просмотр онтологии
    Then система отображает дерево классов
    And система отображает граф онтологии
    And связь "РаботаетВ" видна на графе между "Person" и "Organization"
```
