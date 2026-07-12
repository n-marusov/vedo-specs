<a id="us-git.branches.create-merge"></a>
# US-git.branches.create-merge: Создание ветки и выполнение merge

```gherkin
@US-git.branches.create-merge @UC-git.branches.manage-branch-workflow @P2 @versioning @branch @merge
Feature: US-git.branches.create-merge Создание ветки и выполнение merge

  Background:
    Given пользователь аутентифицирован как "Инженер знаний"
    And открыта онтология "TestOntology"

  Scenario: Инженер знаний создаёт ветку и выполняет merge
    Given ветка "main" содержит классы "Person" и "Organization"
    When пользователь создаёт ветку "feature/new-property" от "main"
    Then активная ветка переключается на "feature/new-property"
    And классы "Person" и "Organization" присутствуют в новой ветке
    When пользователь инициирует merge "feature/new-property" в "main"
    Then система создаёт merge commit или показывает конфликт для ручного разрешения
```
