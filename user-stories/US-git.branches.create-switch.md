<a id="us-git.branches.create-switch"></a>
# US-git.branches.create-switch: Создание и переключение между ветками

```gherkin
@US-git.branches.create-switch @UC-git.branches.manage-branch-workflow @P1 @versioning @branch
Feature: US-git.branches.create-switch Создание и переключение между ветками

  Background:
    Given пользователь аутентифицирован как "Инженер знаний"
    And открыта онтология "TestOntology"
    And существует ветка "main" с head-коммитом "Initial ontology"

  Scenario: Инженер знаний создаёт новую ветку
    Given существует ветка "main"
    When пользователь создаёт ветку "feature/add-vehicle-class"
    And указывает source-ветку "main"
    Then система создаёт ветку "feature/add-vehicle-class"
    And head "feature/add-vehicle-class" совпадает с head "main"
    And ветка отображается в списке веток

  Scenario: Инженер знаний переключается между ветками
    Given существуют ветки "main" и "feature/new-classes"
    And текущая ветка "main"
    When пользователь переключается на ветку "feature/new-classes"
    Then система меняет контекст на "feature/new-classes"
    And дерево классов отображает состояние ветки "feature/new-classes"

  Scenario: Система сохраняет несохранённые изменения при переключении
    Given текущая ветка "main"
    And есть несохранённые изменения
    When пользователь переключается на ветку "feature/new-classes"
    Then система показывает запрос "You have unsaved changes. Commit or discard?"
    And ветка не переключается до подтверждения

  Scenario: Система отклоняет дублирующееся имя ветки
    Given существует ветка "main"
    When пользователь пытается создать ветку "main"
    Then система показывает ошибку "Branch 'main' already exists"
```
