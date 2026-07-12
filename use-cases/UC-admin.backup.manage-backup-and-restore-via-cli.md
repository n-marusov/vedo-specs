### UC-admin.backup.manage-backup-and-restore-via-cli: Управлять backup/restore через `vedo-cli`
**Акторы:** DevOps, Администратор  
**Приоритет:** P0 (критичный)  
**Ключевая функция:** F9.1 — Backup/restore  

**Описание:** Единый CLI-сценарий резервного копирования, проверки и восстановления VEDO Core.
**Основной поток (ежедневный полный backup):**
1. DevOps настраивает schedule/job через `vedo-cli backup schedule` или эквивалентный manifest.
2. `vedo-cli` создаёт full backup: TBox в canonical Turtle, ABox как Neo4j binary dump, Version Store через `pg_dump -Fc`, WAL/incremental logs и LFS-объекты.
3. Backup сохраняется в S3/MinIO-compatible storage с учётом deployment model.
4. `vedo-cli backup verify` проверяет целостность backup.
5. Команда возвращает backup ID и пишет audit log.
**Основной поток (restore):**
1. DevOps выбирает backup ID.
2. Выполняет `vedo-cli restore --id <backup_id>` по RTO-протоколу.
3. Система восстанавливает компоненты в корректном порядке и выполняет post-restore checks.
**Альтернативные потоки:**
- А1: Проверка backup завершилась ошибкой — backup помечается invalid, alert отправляется DevOps.
- А2: Air-gapped environment — используется локальный MinIO/S3-compatible storage без public cloud.
**Постусловия:** Backup проверен и доступен для восстановления либо restore завершён с audit trail.
