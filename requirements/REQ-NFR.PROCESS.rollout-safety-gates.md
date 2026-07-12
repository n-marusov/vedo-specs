# Шлюзы безопасности rollout и auto-rollback

| Атрибут | Значение |
|---------|----------|
| **ID** | REQ-NFR.PROCESS.rollout-safety-gates |
| **Уровень** | NFR |
| **Атрибут качества** | Reliability |
| **Приоритет** | P0 |
| **Статус** | УТВЕРЖДЕНО |
| **Источник** | — |
| **Критерии приёмки** | — |

---

## Что описывает

Фиксирует обязательные блокирующие требования к progressive rollout в production и правила автоматического отката при деградации.

## Область действия

- Все production deployments для API Gateway, Ontology Service, Versioning Service, Auth Service, Metrics Service.
- Все релизы с изменением API, схемы данных, security-контролей, маршрутизации, авторизации.

## Обязательные этапы rollout

| Этап | Доля трафика | Минимальная длительность наблюдения |
|---|---:|---:|
| S0 | 0% (smoke) | 5 минут |
| S1 | 5% | 12 минут |
| S2 | 25% | 12 минут |
| S3 | 50% | 12 минут |
| S4 | 100% | 15 минут |

Пропуск этапов запрещен.

## Hard-threshold блокировки

Переход на следующий этап запрещен, если выполняется хотя бы одно условие:

| Метрика | Условие блокировки | Окно |
|---|---|---|
| HTTP 5xx rate | canary > baseline x 1.5 | 3 минуты |
| Latency p95 | canary > baseline x 1.2 | 5 минут |
| Latency p99 | canary > baseline x 1.3 | 5 минут |
| Error budget burn rate | > 2.0 | 10 минут |
| Success ratio критичных операций | < 99.5% | 5 минут |

## Auto-rollback

- При срабатывании любого hard-threshold должен запускаться auto-rollback.
- Целевое время старта rollback: <= 60 секунд с момента срабатывания условия.
- Полное завершение rollback до предыдущей стабильной версии: <= 2 минут.
- После rollback выполняется обязательный post-rollback smoke check (health + auth + write-path).

## Ограничения ручного вмешательства

- Ручной bypass hard-threshold запрещен.
- Ручной override допускается только по break-glass процедуре с двойным подтверждением (SRE Lead + Security Lead).
- Каждый override должен содержать ticket, причину и время действия (TTL <= 60 минут).

## Evidence и аудит

Для каждого этапа rollout обязательно сохраняются:

- baseline/canary значения метрик;
- решение gate (`pass`/`fail`);
- инициатор и pipeline ID;
- hash артефактов релиза;
- факт auto-rollback (если был) и время восстановления.

Хранение evidence: WORM/append-only не менее 3 лет.

## Бизнес-правила

- Production deployment без progressive stages запрещен.
- Release считается неуспешным, если выполнен rollback на любом этапе S1-S4.
- До повторного релиза обязателен corrective MR с устранением причины rollback.
- Пороговые значения могут быть ужесточены, но не ослаблены без обновления ADR и согласования Architecture Board.
