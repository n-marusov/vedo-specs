### UC-admin.migration.apply-migrations-with-rollback-via-cli: Выполнять миграции с rollback через `vedo-cli`
**Актор:** DevOps  
**Приоритет:** P0 (критичный)  
**Ключевая функция:** F9.2 — Миграции с rollback  

**Описание:** Безопасное применение Neo4j/PostgreSQL миграций с обязательной возможностью отката.
**Основной поток:**
1. DevOps запускает `vedo-cli migrate plan` для просмотра migration map.
2. `vedo-cli` проверяет наличие verified pre-migration backup.
3. DevOps запускает `vedo-cli migrate apply`.
4. `vedo-cli` применяет идемпотентные Neo4j и PostgreSQL scripts.
5. `vedo-cli` выполняет integrity checks: counts, commit history, SPARQL spot checks, hash checks.
6. Результат миграции и версии фиксируются в audit log.
**Альтернативный поток (rollback):**
1. При ошибке DevOps запускает `vedo-cli migrate rollback`.
2. `vedo-cli` останавливает сервисы или переводит их в read-only.
3. `vedo-cli` восстанавливает pre-migration backup и выполняет post-restore verification.
**Постусловия:** Миграция применена и проверена либо выполнен rollback к предыдущему состоянию.
