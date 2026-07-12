<a id="us-admin.migration.diff"></a>
# US-admin.migration.diff: Сравнение двух версий онтологии через diff

```gherkin
@US-admin.migration.diff @UC-admin.migration.transfer-and-diff-ontologies-via-cli @P1 @ontology-admin @diff @vedo-cli
Feature: US-admin.migration.diff Сравнение двух версий онтологии через diff

  Background:
    Given администратор онтологий имеет доступ к двум версиям онтологии

  Scenario: Администратор сравнивает версии онтологии
    Given существует файл "production.ttl"
    And существует файл "staging.ttl"
    When администратор выполняет команду "vedo-cli ontology diff --left production.ttl --right staging.ttl"
    Then vedo-cli показывает добавленные сущности
    And vedo-cli показывает удалённые сущности
    And vedo-cli показывает изменённые сущности
    And неожиданные удаления требуют явного подтверждения перед import в production
```
