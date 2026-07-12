# ADR-DES.PROCESS.rollback-data-strategy

**Дата:** 2026-05-10  
**Статус:** Принято

## Контекст

Rollback major version в VEDO Core может затрагивать не только код, но и данные: TBox/ABox в Neo4j, Version Store в PostgreSQL, LFS-объекты и метаданные версионирования. Если новая версия изменила схему БД или формат данных, простой rollback приложения на предыдущий container image не гарантирует работоспособность и может оставить данные в несовместимом состоянии.

## Требование-источник
- [migration-rollback.md](requirements/REQ-FUN.DATA.migration-rollback.md)
- [rollback-data-policy.md](requirements/REQ-NFR.DATA.rollback-data-policy.md)
- [rollback-runbook.md](requirements/REQ-NFR.DATA.rollback-runbook.md)
- [rollback-data-loss-rpo.md](requirements/REQ-NFR.DATA.rollback-data-loss-rpo.md)
- [partial-data-rollback.md](requirements/REQ-NFR.DATA.partial-data-rollback.md)
- [rollback-prevention.md](requirements/REQ-NFR.DATA.rollback-prevention.md)

## Решение

Принять двухуровневую стратегию rollback пользовательских данных: rollback кода без изменения данных при неизменной схеме БД и восстановление из pre-migration backup при изменении схемы в major версии.

Явная политика rollback даёт предсказуемые RTO/RPO ожидания вместо best-effort поведения. Быстрый rollback (60 минут, потеря данных до 60 минут работы пользователей) минимизирует простой для критических инцидентов. Поздний rollback (после 24 часов) не выполняется автоматически, а требует консультации с заказчиком, что предотвращает несогласованную потерю данных.

Процедура включает read-only mode, завершение write operations, остановку приложения, восстановление из pre-migration backup при необходимости, проверку целостности и уведомление пользователей. Rollback не выполнять автоматически при > 1000 новых объектов или > 48 часов после upgrade.

## Рассмотренные альтернативы

| Альтернатива | Причина отклонения |
|--------------|--------------------|
| Всегда откатывать только код | Небезопасно при изменении схемы БД и формата данных |
| Всегда восстанавливать данные из backup | Избыточно для minor/patch rollback без изменения схемы и ведет к ненужной потере данных |
| Поддерживать полный rollback через несколько дней без потери данных | Требует сложной двунаправленной миграции и reconciliation, риск выше пользы |
| Выполнять rollback автоматически при P0 | Может привести к несогласованной потере пользовательских данных без решения заказчика |
| Игнорировать частичный rollback | Потеря возможности исправлять data-only ошибки без отката версии ПО |

## Последствия

**Положительные последствия:**
- Пользовательские данные имеют явную политику при rollback, а не best-effort поведение.
- Быстрый rollback имеет понятные RTO/RPO ожидания.
- Поздний rollback не выполняется автоматически и требует консультации с заказчиком.
- Частичный rollback через Git-like commits снижает потребность в полном откате версии.

**Отрицательные последствия:**
- При быстром rollback major version возможна потеря до 60 минут пользовательской работы.
- После 24 часов full rollback практически невозможен без потери данных.
- Runbook требует строгой изоляции write operations и корректных pre-migration backups.

**Меры снижения рисков:**
- Перед major upgrade обязательно выполнять pre-migration backup и sandbox migration.
- Использовать read-only mode перед восстановлением данных.
- Применять флаги возможностей, canary развёртывание, shadow mode и Grafana dashboards для раннего обнаружения проблем.
- Уведомлять пользователей о backup timestamp, upgrade timestamp, rollback timestamp и возможной потере изменений.
- Для data-only ошибок применять частичный rollback на уровне коммитов.

---
