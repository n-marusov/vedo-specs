# Runbook rollback версии

| Атрибут | Значение |
|---------|----------|
| **ID** | REQ-NFR.DATA.rollback-runbook |
| **Уровень** | NFR |
| **Атрибут качества** | Supportability |
| **Приоритет** | P1 |
| **Статус** | УТВЕРЖДЕНО |
| **Источник** | — |
| **Критерии приёмки** | — |

---

## Шаг 0: Решение о rollback

Rollback допускается при следующих условиях:

- Критический баг, блокирующий работу (P0).
- Потеря данных, не исправляемая срочным исправлением (hotfix).
- Уязвимость безопасности, не закрываемая патчем.

Rollback не выполняется автоматически, если:

- Пользователи создали больше 1000 новых объектов.
- С момента обновления прошло больше 48 часов.

## Шаг 1: Остановка сервисов и изоляция данных

1. Перевести API Gateway в read-only mode.
2. Дождаться завершения write operations, максимум 5 минут.
3. Остановить приложение.

```bash
kubectl patch configmap vedo-config --patch '{"data":{"READ_ONLY":"true"}}'
kubectl rollout status deployment/vedo-core --timeout=300s
kubectl scale deployment vedo-core --replicas=0
```

## Шаг 2: Восстановление баз данных из pre-migration backup

```bash
neo4j-admin load --from=/backup/pre_migration/abox.dump --database=neo4j --force
pg_restore -d vedo_version /backup/pre_migration/version_store.dump
mc mirror /backup/pre_migration/lfs/ minio/vedo-lfs/
```

LFS заменяется полностью только если это требуется сценарием rollback.

## Шаг 3: Базовая проверка целостности

Neo4j:

```cypher
MATCH (n) RETURN count(n) AS node_count;
```

PostgreSQL:

```sql
SELECT COUNT(*) FROM commits;
```

Значения должны совпадать с данными в логе миграции.

## Шаг 4: Запуск v1.0 и health check

```bash
kubectl scale deployment vedo-core --replicas=3
kubectl rollout status deployment/vedo-core
curl -s http://vedo-core/health | jq -e '.status == "ok"'
```

## Шаг 5: Уведомление пользователей

Сообщение пользователям должно содержать:

- Факт rollback на предыдущую версию.
- Timestamp backup, из которого восстановлены данные.
- Upgrade timestamp и rollback timestamp.
- Предупреждение: изменения между upgrade и rollback могли быть потеряны.
- Канал обращения в support при расхождении данных.

Каналы уведомления:

- Email администраторам онтологий.
- Баннер в интерфейсе.
- Status page.

## Открытые вопросы

- Нет по M2.3.
