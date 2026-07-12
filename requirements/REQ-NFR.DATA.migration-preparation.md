# Подготовка миграции мажорной версии

| Атрибут | Значение |
|---------|----------|
| **ID** | REQ-NFR.DATA.migration-preparation |
| **Уровень** | NFR |
| **Атрибут качества** | Supportability |
| **Приоритет** | P1 |
| **Статус** | УТВЕРЖДЕНО |
| **Источник** | — |
| **Критерии приёмки** | — |

---

## Карта миграции (Migration Map)

Перед миграцией создается документ «Migration Map» — что меняется, что мигрируется, что удаляется.

Оцениваются:

- Схема Neo4j (TBox, ABox): сравнение моделей (diff), ответственный DBA / DevOps.
- Схема PostgreSQL (Version Store): сравнение SQL-дампов, ответственный DBA.
- Формат коммитов (дельта JSON): JSON schema diff, ответственный backend-разработчик.
- API endpoints (REST, GraphQL): Swagger diff, ответственный backend-разработчик.
- Формат экспорта (Turtle, RDF/XML): проверка на совместимость, ответственный backend-разработчик.

## Полный бэкап перед миграцией

Полный бэкап включает:

- TBox: канонический Turtle.
- ABox: Neo4j dump.
- Version Store: PostgreSQL dump.
- Аудиторские логи: Elastic snapshot.
- LFS объекты: MinIO -> backup.

Пример runbook для бэкапа:

```bash
docker exec neo4j bash -c "cypher-shell 'MATCH (n) RETURN n' --format ttl" > /backup/pre_migration/tbox.ttl
docker exec neo4j neo4j-admin dump --database=neo4j --to=/backup/pre_migration/abox.dump
pg_dump -Fc vedo_version > /backup/pre_migration/version_store.dump
curl -X PUT "localhost:9200/_snapshot/vedo_backup/pre_migration_snapshot"
mc mirror --watch minio/vedo-lfs/ /backup/pre_migration/lfs/
```

## Тестовая миграция в изолированной среде

Цель: проверить процесс миграции и оценить время даунтайма.

Процесс:

1. Развернуть копию production в изолированном окружении.
2. Запустить миграцию `migrate-major.sh` из версии N в версию N+1.
3. Замерить время выполнения.
4. Зафиксировать лог миграции и отчет о времени даунтайма.

Требование по времени: даунтайм должен быть не больше 2 часов.

## Открытые вопросы

- Нет по M2.1.
