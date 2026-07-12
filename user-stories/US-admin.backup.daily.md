<a id="us-admin.backup.daily"></a>
# US-admin.backup.daily: Ежедневный автоматический backup всей онтологии

```gherkin
@US-admin.backup.daily @UC-admin.backup.manage-backup-and-restore-via-cli @P0 @admin @backup @vedo-cli
Feature: US-admin.backup.daily Ежедневный автоматический backup всей онтологии

  Background:
    Given DevOps имеет административные права на окружение "production"
    And настроено S3/MinIO-compatible storage

  Scenario: DevOps создаёт и проверяет полный backup
    When DevOps выполняет команду "vedo-cli backup create --full --name pre-upgrade"
    Then создаётся backup в S3/MinIO-compatible storage
    And backup включает TBox в canonical Turtle
    And backup включает ABox как Neo4j binary dump
    And backup включает WAL или incremental logs для Point-in-Time Recovery
    And команда возвращает ID backup
    When DevOps выполняет проверку backup по ID
    Then проверка подтверждает целостность backup
    And действие записано в audit log
```
