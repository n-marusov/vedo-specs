<a id="us-admin.migration.schema-rollback"></a>
# US-admin.migration.schema-rollback: Миграция схемы Neo4j/PostgreSQL с rollback

```gherkin
@US-admin.migration.schema-rollback @UC-admin.migration.apply-migrations-with-rollback-via-cli @P0 @admin @migration @rollback @vedo-cli
Feature: US-admin.migration.schema-rollback Миграция схемы Neo4j/PostgreSQL с rollback

  Background:
    Given DevOps имеет административные права на окружение "production"
    And существует verified pre-migration backup

  Scenario: DevOps выполняет миграцию и откатывает её при ошибке
    Given подготовлены идемпотентные миграции Neo4j и PostgreSQL
    When DevOps выполняет команду "vedo-cli migrate apply --version 2.0.0"
    Then система применяет миграции Neo4j и PostgreSQL
    And система выполняет integrity checks после миграции
    And результат фиксируется в audit log
    When проверка миграции завершается ошибкой
    Then DevOps выполняет команду "vedo-cli migrate rollback --to pre-migration"
    And система восстанавливается из pre-migration backup
    And post-restore verification подтверждает целостность
```
