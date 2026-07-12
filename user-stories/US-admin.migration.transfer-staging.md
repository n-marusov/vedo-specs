<a id="us-admin.migration.transfer-staging"></a>
# US-admin.migration.transfer-staging: Экспорт production ontology в staging

```gherkin
@US-admin.migration.transfer-staging @UC-admin.migration.transfer-and-diff-ontologies-via-cli @P1 @ontology-admin @import-export @vedo-cli
Feature: US-admin.migration.transfer-staging Экспорт production ontology в staging

  Background:
    Given администратор онтологий имеет доступ к окружениям "production" и "staging"

  Scenario: Администратор переносит ontology из production в staging
    When администратор выполняет команду "vedo-cli ontology export --env production --format turtle --canonical --output ontology.ttl"
    Then vedo-cli создаёт canonical Turtle export
    And production остаётся неизменным
    When администратор выполняет команду "vedo-cli ontology import --env staging --input ontology.ttl"
    Then ontology загружена в staging
    And изменения не влияют на живую production-систему
```
