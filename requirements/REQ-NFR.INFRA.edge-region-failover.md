# Edge & Regional Resilience Specification — Technical Requirements v1.0

| Атрибут | Значение |
|---------|----------|
| **ID** | REQ-NFR.INFRA.edge-region-failover |
| **Уровень** | NFR |
| **Атрибут качества** | Reliability |
| **Приоритет** | P1 |
| **Статус** | УТВЕРЖДЕНО |
| **Источник** | — |
| **Критерии приёмки** | — |

---

## Обзор

Документ определяет требования к отказоустойчивости граничных компонентов (CDN, WAF, Edge-вычисления) и кросс-региональной архитектуре VEDO Core. Регламентирует целевые RTO/RPO, режимы деградации, процедуры failover и rollback, мониторинг, chaos engineering и особенности on-premise/air-gapped развёртываний.

Политика распространяется на три модели развёртывания:
- **SaaS (облачный)** — мультирегиональная архитектура с CDN/WAF/Edge у конкретного cloud-провайдера
- **On-premise** — единый регион, CDN/WAF опциональны, failover на уровне Kubernetes/Helm
- **Air-gapped** — полностью изолированный контур, CDN/WAF/Edge отсутствуют, отказоустойчивость на уровне кластера

---

## 1. Целевые RTO/RPO

### 1.1 Edge-компоненты

| Компонент | RTO | RPO | Тип failover | Автоматизация |
|-----------|-----|-----|-------------|---------------|
| CDN (CloudFront / Cloudflare / Yandex CDN) | 5 мин | 0 | Автоматический к резервному CDN-провайдеру | Полностью автоматический |
| WAF (AWS WAF / Cloudflare WAF / Yandex WAF) | 10 мин | 0 | Автоматическое переключение в режим monitor-only/bypass | Полностью автоматический |
| Edge-вычисления (Workers / Lambda@Edge) | 15 мин | 0 | Fallback к региональному API Gateway | Автоматический |
| API Gateway (один инстанс) | 10 мин | 0 | Read-only или переключение на standby | Частично автоматический (manual switch) |

### 1.2 Региональные компоненты

| Компонент | RTO | RPO | Тип failover | Автоматизация |
|-----------|-----|-----|-------------|---------------|
| Отказ региона А (полный) | 30 мин | 15 мин | Ручной failover на регион Б (warm standby) | Manual approval SRE |
| Neo4j (асинхронная репликация) | 30 мин | ≤ 15 мин | Промоция read replica в primary | Частично автоматический |
| PostgreSQL (streaming replication) | 15 мин | ≤ 5 мин | Промоция hot standby | Частично автоматический |
| Redis (cache + locks) | 15 мин | ≤ 5 мин | Разогрев кэша в регионе Б | Автоматический при failover |
| RabbitMQ (очереди) | 30 мин | ≤ 15 мин | Восстановление из persisted queue в регионе Б | Ручной |

---

## 2. Режимы деградации

### 2.1 CDN bypass

| Параметр | Значение |
|----------|----------|
| **Триггер** | CDN недоступен > 30 сек, error rate > 5% |
| **Режим** | Статические ассеты (JS, CSS, изображения, шрифты) идут напрямую из API Gateway |
| **Доступные функции** | Все — только с увеличенной задержкой загрузки статики |
| **Недоступные функции** | Нет |
| **Влияние на пользователя** | Увеличение времени загрузки страницы (статический контент без кэша CDN) |
| **Возврат к норме** | Автоматический при восстановлении CDN (health check + 3 успешных probing) |
| **P0 алерт** | Да — CDN недоступен |

### 2.2 WAF bypass

| Параметр | Значение |
|----------|----------|
| **Триггер** | WAF недоступен > 60 сек, error rate > 10% |
| **Режим** | Сервис работает без WAF с P0 алертом; включена baseline фильтрация на API Gateway (IP rate limiting, basic input sanitization, JWT-валидация) |
| **Доступные функции** | Все — с минимальной защитой на уровне API Gateway |
| **Недоступные функции** | WAF-specific: advanced rule sets, bot detection, custom threat intelligence |
| **Влияние на пользователя** | Повышенный (но контролируемый) риск эксплуатации веб-уязвимостей на время отказа |
| **Возврат к норме** | Автоматический при восстановлении WAF + синхронизация правил |
| **P0 алерт** | Да — WAF недоступен, включён bypass-режим |

### 2.3 Edge fallback

| Параметр | Значение |
|----------|----------|
| **Триггер** | Edge-вычисления недоступны > 30 сек |
| **Режим** | Rate limiting и JWT-валидация переносятся на региональный API Gateway |
| **Доступные функции** | Все — rate limiting и JWT-валидация работают на API Gateway |
| **Недоступные функции** | Edge-specific: A/B-тестирование на границе, request transformation на edge, geo-routing |
| **Влияние на пользователя** | Незначительное (дополнительная нагрузка на API Gateway) |
| **Возврат к норме** | Автоматический при восстановлении Edge-вычислений |
| **P0 алерт** | Да — Edge-вычисления недоступны |

### 2.4 API Gateway (один инстанс)

| Параметр | Значение |
|----------|----------|
| **Триггер** | Health check API Gateway провален > 10 сек, replica count < desired |
| **Режим** | Read-only или переключение на standby инстанс |
| **Доступные функции** | Чтение онтологий, поиск, просмотр истории (read-only). Write-операции очередируются или отклоняются с понятным сообщением |
| **Недоступные функции** | Создание/редактирование классов, свойств, индивидов; коммиты; merge request; импорт/экспорт |
| **Влияние на пользователя** | Режим «только чтение» до восстановления write capacity |
| **Возврат к норме** | Kubernetes auto-recovery или ручной перезапуск pod |
| **P1 алерт** | Да — API Gateway degraded |

### 2.5 Отказ региона А (полный)

| Параметр | Значение |
|----------|----------|
| **Триггер** | Потеря связности с регионом А > 5 мин, множественные P0 алерты по критическим сервисам |
| **Режим** | Warm standby регион Б с read-only репликами → после failover полная функциональность |
| **Доступные функции (до failover)** | Чтение онтологий (через read replica региона Б), просмотр истории |
| **Доступные функции (после failover)** | Все — с учётом времени разогрева кэша и продвижения реплик |
| **Недоступные функции (до failover)** | Запись, коммиты, импорт, изменение прав |
| **Влияние на пользователя** | См. раздел 4.4 — User Experience During Failover |
| **Возврат к норме** | Ручной rollback после полного восстановления региона А (см. раздел 5) |
| **P0 алерт** | Да — критический инцидент с эскалацией по P0 escalation matrix |

---

## 3. Multi-Region архитектура

### 3.1 Топология

```
                        ┌──────────────────────┐
                        │       DNS / Route53    │
                        │   (Latency-based /     │
                        │    Geo-routing)        │
                        └──────┬───────────────┘
                               │
              ┌────────────────┼────────────────┐
              │                                 │
    ┌─────────▼──────────┐         ┌───────────▼──────────┐
    │   Регион А (Active) │         │  Регион Б (Warm)     │
    │   read/write        │         │  read-only (до       │
    │                     │         │  failover)           │
    │  ┌───────────────┐  │         │  ┌───────────────┐   │
    │  │  CDN + WAF +  │  │         │  │  CDN + WAF +  │   │
    │  │  Edge         │  │         │  │  Edge         │   │
    │  └───────┬───────┘  │         │  └───────┬───────┘   │
    │          │          │         │          │           │
    │  ┌───────▼───────┐  │         │  ┌───────▼───────┐   │
    │  │ API Gateway   │  │         │  │ API Gateway   │   │
    │  │ (Rust/Go)     │  │         │  │ (Rust/Go)     │   │
    │  └───────┬───────┘  │         │  └───────┬───────┘   │
    │          │          │         │          │           │
    │  ┌───────▼───────┐  │         │  ┌───────▼───────┐   │
    │  │  Ontology     │  │         │  │  Ontology     │   │
    │  │  Versioning   │  │   ◄─────┼──┤  Versioning   │   │
    │  │  Auth         │  │ Async   │  │  Auth         │   │
    │  │  Metrics      │  │  repl.  │  │  Metrics      │   │
    │  └───────┬───────┘  │         │  └───────┬───────┘   │
    │          │          │         │          │           │
    │  ┌───────▼───────┐  │         │  ┌───────▼───────┐   │
    │  │  Neo4j RW     │──┼─────────┼─►│  Neo4j RO     │   │
    │  │  PostgreSQL RW│──┼─────────┼─►│  PostgreSQL RO│   │
    │  │  Redis RW     │  │         │  │  Redis (empty)│   │
    │  │  RabbitMQ     │  │         │  │  RabbitMQ     │   │
    │  └───────────────┘  │         │  └───────────────┘   │
    └─────────────────────┘         └──────────────────────┘
```

### 3.2 Конфигурация региона

**Регион А (активный):**

| Компонент | Конфигурация | Режим |
|-----------|-------------|-------|
| CDN | Основной провайдер (CloudFront / Cloudflare / Yandex CDN) | Активный |
| WAF | Полный набор правил (OWASP Top 10, custom rules, bot detection) | Активный (enforce) |
| Edge-вычисления | Workers / Lambda@Edge для rate limiting, JWT-валидации, A/B-тестирования | Активные |
| API Gateway | 3+ реплики в разных availability zones | Активный |
| Backend services | 2+ реплики каждого сервиса | Активные read/write |
| Neo4j | Primary кластер (3+ узла) | Read/write |
| PostgreSQL | Primary с streaming replication к региону Б | Read/write |
| Redis | Primary cluster (3+ узла) | Read/write, session store, locks |
| RabbitMQ | Primary cluster (3+ узла) | Read/write |

**Регион Б (warm standby):**

| Компонент | Конфигурация | Режим |
|-----------|-------------|-------|
| CDN | Резервный провайдер | Холодный резерв |
| WAF | Полный набор правил (синхронизирован с регионом А) | Холодный резерв |
| Edge-вычисления | Те же функции + fallback конфигурация | Ожидание |
| API Gateway | 2+ реплики minimal | Дежурный режим (duty), health-check активен |
| Backend services | 1 реплика каждого сервиса | Дежурный режим |
| Neo4j | Read replica (асинхронная репликация, RPO ≤ 15 мин) | Read-only |
| PostgreSQL | Hot standby (streaming replication, RPO ≤ 5 мин) | Read-only |
| Redis | Отключён (разогрев после failover) | Ожидание |
| RabbitMQ | Отключён (восстановление после failover) | Ожидание |

### 3.3 Репликация данных

| Хранилище | Метод репликации | RPO | Задержка (типичная) | Асинхронность |
|-----------|-----------------|-----|---------------------|---------------|
| Neo4j | Асинхронная кросс-региональная репликация (Causal clustering) | ≤ 15 мин | 1–10 сек | Асинхронная |
| PostgreSQL | Асинхронная streaming replication (primary → hot standby) | ≤ 5 мин | 0.1–2 сек | Асинхронная |
| Redis | Cross-region replication через Redis Enterprise CRDT или Active-Passive | ≤ 5 мин | 0.5–5 сек | Асинхронная |
| RabbitMQ | Shovel / Federation upstream для кросс-региональной очереди | ≤ 15 мин | 1–10 сек | Асинхронная |
| S3/MinIO (LFS-объекты) | Cross-region replication (S3 CRR / MinIO bucket replication) | ≤ 15 мин | 1–10 мин | Асинхронная |

### 3.4 Требования к регионам

| Требование | Регион А | Регион Б |
|------------|----------|----------|
| Минимальное расстояние | — | ≥ 500 км (защита от geo-катастроф) |
| Cloud-провайдер | Yandex Cloud / AWS | Любой (может отличаться от региона А) |
| Задержка между регионами | — | ≤ 50 мс (p95) для целей репликации |
| Kubernetes кластер | K8s ≥ 1.28, 3+ worker nodes | K8s ≥ 1.28, 2+ worker nodes |
| Container Registry | GitLab Registry + mirror в регионе Б | GitLab Registry (локальный кэш) |
| Helm chart | Единый — различие только в values-профилях | Единый — те же values, что регион А (overrides для read-only) |
| Сертификаты TLS | Wildcard `*.vedo.dev` или регионарные | Wildcard `*.vedo.dev` или регионарные |

### 3.5 Multi-cloud / Multi-провайдер

VEDO Core может быть развёрнут у разных cloud-провайдеров для региона А и региона Б. В этом случае действуют дополнительные требования:

- **DNS-балансировка:** Route53 (AWS) или эквивалентный DNS-сервис с latency-based routing
- **Container Registry:** GitLab Registry или Harbour с кросспровайдерной репликацией
- **Kubernetes:** EKS (AWS) / Managed Kubernetes (Yandex) / vanilla K8s (on-premise) — без привязки к cloud-specific API
- **Network:** Site-to-site VPN или PrivateLink между провайдерами для replication traffic
- **IAM:** Раздельные identity domains с federation через OIDC

---

## 4. Процедура Failover

### 4.1 Failover Decision Matrix

Failover региона А → региона Б выполняется только при одновременном выполнении **всех** условий:

| Условие | Порог | Критичность |
|---------|-------|-------------|
| Потеря связности с регионом A | > 5 минут | CRITICAL |
| NEO4j primary недоступен | > 3 минут и не восстанавливается | CRITICAL |
| PostgreSQL primary недоступен | > 3 минут и не восстанавливается | CRITICAL |
| API Gateway error rate региона А | > 50% в течение 5 минут | HIGH |
| Backend services (Ontology + Versioning) | ≥ 2 из 4 сервисов недоступны > 5 минут | CRITICAL |
| Потеря кросс-региональной репликации | > 30 минут без подтверждения восстановления | HIGH |

**False positive prevention** — failover НЕ выполняется, если:

| Условие | Обоснование |
|---------|-------------|
| Единичный сбой CDN без потери региона | CDN bypass (RTO 5 мин) покрывает сценарий |
| WAF error rate spike | WAF bypass (RTO 10 мин) покрывает сценарий |
| API Gateway degraded, но backend alive | API Gateway read-only mode (RTO 10 мин) покрывает |
| Кратковременная потеря связности (< 5 мин) | Кратковременные сбои не стоят риска полного переключения региона |
| Плановая миграция (maintenance window) | Failover при плановых работах не выполняется; используется blue-green в рамках региона |
| Регион Б не прошёл pre-flight check | Failover на неготовый регион опаснее, чем degraded режим в регионе А |

### 4.2 Pre-flight проверки перед failover

Перед инициацией failover SRE выполняет обязательные pre-flight checks:

```bash
# 1. Проверка доступности региона Б
kubectl --context=region-b get nodes --show-labels | grep -q "Ready"
kubectl --context=region-b get pods --all-namespaces | grep -v Running | grep -v Completed | wc -l  # ожидаем 0

# 2. Проверка репликации
# PostgreSQL: lag в байтах
psql -h region-b-pg-replica -c "SELECT pg_wal_lsn_diff(pg_stat_replication.replay_lsn, pg_stat_replication.sent_lsn) AS lag_bytes;"

# Neo4j: lag в транзакциях
neo4j-admin check --region=region-b replication-lag

# 3. Проверка DNS
dig region-b.api.vedo.dev +short | head -1  # должен быть непустым

# 4. Проверка сертификатов
openssl s_client -connect region-b.api.vedo.dev:443 -servername region-b.api.vedo.dev < /dev/null 2>/dev/null | openssl x509 -noout -dates

# 5. Проверка readiness API Gateway региона Б
curl -f https://region-b.api.vedo.dev/health

# 6. Проверка S3/MinIO репликации
aws s3api list-objects --bucket vedo-lfs --region region-b --max-items 1 > /dev/null
```

### 4.3 Пошаговая процедура failover

**Фаза 0: Обнаружение (0–5 мин)**

| Шаг | Действие | Исполнитель | Time budget |
|-----|----------|-------------|-------------|
| 0.1 | P0 алерт получен (multiple P0) | Alertmanager | 0 мин |
| 0.2 | SRE подтверждает инцидент (Incident Commander) | SRE (дежурный) | ≤ 5 мин |
| 0.3 | Запуск диагностики `vedo-cli diagnose region --region A` | SRE | ≤ 5 мин |
| 0.4 | Оценка по failover decision matrix | Incident Commander | ≤ 5 мин |

**Фаза 1: Pre-flight (5–10 мин)**

| Шаг | Действие | Исполнитель | Time budget |
|-----|----------|-------------|-------------|
| 1.1 | Pre-flight checks региона Б (все 6) | SRE | ≤ 5 мин |
| 1.2 | Communication Lead уведомляет stakeholders | Comms Lead | ≤ 2 мин |
| 1.3 | Если pre-flight провален — abort failover, escalation L4 | Incident Commander | ≤ 2 мин |

**Фаза 2: Execution (10–20 мин)**

| Шаг | Действие | Исполнитель | Time budget |
|-----|----------|-------------|-------------|
| 2.1 | Регион А переводится в read-only / drain mode | SRE | ≤ 2 мин |
| 2.2 | PostgreSQL: promote hot standby → primary | SRE | ≤ 2 мин |
| 2.3 | Neo4j: promote read replica → primary | SRE | ≤ 3 мин |
| 2.4 | API Gateway региона Б: включение write mode | SRE (helm upgrade --set global.readOnly=false) | ≤ 2 мин |
| 2.5 | Backend services региона Б: scale up до полного профиля | SRE (helm upgrade --replicas) | ≤ 3 мин |
| 2.6 | Redis: разогрев кэша (warmup) | Автоматически (vedo-cli cache warmup) | ≤ 5 мин |
| 2.7 | RabbitMQ: восстановление очередей из persisted storage | SRE | ≤ 5 мин |
| 2.8 | DNS: переключение трафика на регион Б (CNAME / ALIAS update) | SRE | ≤ 2 мин |

**Фаза 3: Validation (20–30 мин)**

| Шаг | Действие | Исполнитель | Time budget |
|-----|----------|-------------|-------------|
| 3.1 | Smoke test региона Б (read + write) | SRE / Automated | ≤ 5 мин |
| 3.2 | Проверка всех health endpoints | Автоматически (Grafana + Prometheus) | ≤ 2 мин |
| 3.3 | Проверка целостности данных (spot checks) | SRE | ≤ 5 мин |
| 3.4 | Проверка метрик (error rate, latency, throughput) | SRE (Grafana) | ≤ 3 мин |
| 3.5 | Incident Commander объявляет failover успешным | Incident Commander | — |
| 3.6 | Communication Lead уведомляет stakeholders | Comms Lead | ≤ 2 мин |

### 4.4 User Experience During Failover

| Фаза | Что видит пользователь | Ожидаемое поведение |
|------|-----------------------|---------------------|
| Обнаружение (0–5 мин) | HTTP 503 / timeout на часть запросов | Автоматические retry на клиенте (Apollo Client) |
| Pre-flight (5–10 мин) | 503 / «Service Unavailable. Retrying...» | Retry с exponential backoff |
| Execution (10–20 мин) | DNS propagation — часть пользователей в read-only | Интерфейс в read-only режиме (согласно fault-tolerance-strategy) |
| Validation (20–30 мин) | Постепенное восстановление полной функциональности | Apollo Client refetch queries, UI update |
| После failover | Полная функциональность | Стандартное поведение (возможна повышенная latency из-за разогрева кэша) |

**UI-индикация:** При активации read-only режима (фаза 2.1) интерфейс VEDO Core показывает баннер:
> «Система временно работает в режиме чтения. Ведутся восстановительные работы. Изменения не могут быть сохранены.»

---

## 5. Процедура Rollback

После восстановления региона А выполняется обратное переключение (rollback). Rollback — **ручная процедура** с отдельным окном.

### 5.1 Условия для rollback

- Регион А полностью восстановлен (health checks OK)
- Репликация данных из региона Б в регион А восстановлена (catch-up завершён)
- Окно rollback согласовано с заказчиком / stakeholders (≥ 60 мин downtime)
- Pre-flight checks региона А пройдены

### 5.2 Пошаговая процедура rollback

**Фаза 0: Pre-rollback (за 24–48 часов до)**

| Шаг | Действие | Исполнитель |
|-----|----------|-------------|
| 0.1 | Синхронизация схемы Neo4j/PostgreSQL между регионами (нет расхождений) | SRE |
| 0.2 | Восстановление streaming replication PostgreSQL регион Б → регион А | SRE |
| 0.3 | Восстановление асинхронной репликации Neo4j регион Б → регион А | SRE |
| 0.4 | Разогрев кэша Redis в регионе А (pre-warm) | Автоматически |
| 0.5 | Уведомление stakeholders о запланированном окне rollback | Comms Lead |

**Фаза 1: Execution (0–30 мин)**

| Шаг | Действие | Исполнитель | Time budget |
|-----|----------|-------------|-------------|
| 1.1 | Регион Б переводится в read-only (drain) | SRE | ≤ 2 мин |
| 1.2 | PostgreSQL: ожидание replay всех WAL на регионе А | SRE | ≤ 5 мин |
| 1.3 | Neo4j: catch-up реплика до полной синхронизации | SRE | ≤ 5 мин |
| 1.4 | DNS переключение на регион А | SRE | ≤ 2 мин |
| 1.5 | API Gateway региона А: включение full write mode | SRE | ≤ 2 мин |
| 1.6 | Smoke test региона А (read + write) | SRE | ≤ 5 мин |
| 1.7 | Регион Б возвращается в warm standby (read-only) | SRE | ≤ 5 мин |

### 5.3 Criteria abort rollback

Rollback отменяется, если:

- Smoke test региона А провален (error rate > 1%)
- Данные региона А не синхронизированы с регионом Б (RPO превышен)
- Заказчик отзывает approval

При отмене регион Б остаётся активным, регион А переводится в диагностический режим.

---

## 6. Мониторинг и алерты

### 6.1 Edge-компоненты

| Компонент | Метрика | Порог | Severity | Действие |
|-----------|---------|-------|----------|----------|
| CDN availability | `vedo_cdn_health` | < 99% за 1 мин | P0 | CDN bypass |
| CDN latency | `vedo_cdn_latency_p95` | > 500 мс | P2 | Диагностика |
| WAF availability | `vedo_waf_health` | Недоступен > 60 сек | P0 | WAF bypass |
| WAF blocked requests | `vedo_waf_blocked_rate` | > 10% от общего трафика | P1 | Review ruleset |
| Edge Workers | `vedo_edge_health` | Error rate > 5% | P0 | Edge fallback |
| Edge Workers latency | `vedo_edge_latency_p95` | > 200 мс | P2 | Оптимизация |

### 6.2 API Gateway и backend

| Компонент | Метрика | Порог | Severity | Действие |
|-----------|---------|-------|----------|----------|
| API Gateway availability | `vedo_gateway_up` | 0 (недоступен) | P0 | Read-only / standby |
| API Gateway latency | `vedo_gateway_latency_p99` | > 1 сек | P1 | Scale up / диагностика |
| API Gateway error rate | `vedo_gateway_errors_total` | > 1% | P1 | Анализ ошибок |
| Backend services | `vedo_service_up{service="..."}` | 0 (недоступен) | P0 | Circuit breaker / alert |
| Backend services latency | `vedo_service_latency_p99{service="..."}` | > 2 сек | P1 | Scale up |
| Backend services error rate | `vedo_service_errors{service="..."}` | > 5% | P1 | Диагностика |

### 6.3 Региональные метрики

| Метрика | Порог | Severity | Действие |
|---------|-------|----------|----------|
| Cross-region replication lag (Neo4j) | > 15 мин | P0 | Failover review |
| Cross-region replication lag (Neo4j) | > 5 мин | P1 | Investigate replication |
| PostgreSQL replication lag | > 5 мин | P0 | Failover review |
| PostgreSQL replication lag | > 2 мин | P1 | Investigate replication |
| Region A availability | < 99.9% за 5 мин | P0 | Region failover consideration |
| Region B health check failed | > 30 сек | P1 | SRE notified — регион Б не готов к failover |
| DNS failover switch time | > 5 мин | P1 | DNS provider tuning |
| Redis cache hit rate (после failover) | < 50% | P2 | Разогрев кэша |

### 6.4 Пользовательские SLO

| SLO | Цель | Окно измерения | Severity при нарушении |
|-----|------|----------------|----------------------|
| Availability API calls | ≥ 99.9% | 30 дней | P0 |
| Availability read-only API calls при деградации | ≥ 99.5% | 30 дней | P1 |
| CDN availability (static assets) | ≥ 99.99% | 30 дней | P0 |
| Восстановление после CDN отказа | ≤ 5 мин | Каждый инцидент | P0 |
| Восстановление после регионального отказа | ≤ 30 мин | Каждый инцидент | P0 |
| Потеря данных при региональном отказе | ≤ 15 мин (RPO) | Каждый инцидент | P0 |

---

## 7. Chaos Engineering

### 7.1 Сценарии тестирования

| ID | Сценарий | Описание | Ожидаемый результат | Частота |
|----|----------|----------|--------------------|---------|
| CE-001 | Отказ CDN | Блокировка трафика к основному CDN-провайдеру | Автоматический failover к резервному CDN, RTO ≤ 5 мин | 1 раз в месяц |
| CE-002 | Отказ WAF | Блокировка WAF endpoint | WAF bypass, baseline фильтрация на API Gateway, RTO ≤ 10 мин | 1 раз в месяц |
| CE-003 | Отказ Edge Workers | Остановка всех Workers | Edge fallback к API Gateway, RTO ≤ 15 мин | 1 раз в 2 недели |
| CE-004 | Отказ API Gateway | Убийство 2 из 3 реплик API Gateway | Read-only режим, RTO ≤ 10 мин | 1 раз в 2 недели |
| CE-005 | Отказ Neo4j primary | Stop Neo4j primary node | Circuit breaker, read-only режим; failover при превышении RPO | 1 раз в квартал |
| CE-006 | Потеря связности региона А | Блокировка всего трафика к региону А | Region failover, RTO ≤ 30 мин, RPO ≤ 15 мин | 1 раз в квартал |
| CE-007 | Частичная потеря региона А | Отключение 50% сервисов в регионе А | Graceful degradation — сервисы переходят в read-only | 1 раз в месяц |
| CE-008 | Задержка между регионами | Искусственная задержка 200 мс между регионами А и Б | Наблюдение за replication lag, алерты при превышении RPO | 1 раз в квартал |
| CE-009 | Нагрузочное тестирование региона Б | Симулирование production-нагрузки на регион Б | Регион Б выдерживает нагрузку, latency в пределах SLO | 1 раз в 2 месяца |
| CE-010 | Rollback после failover | Failover → работа на регионе Б → rollback на регион А | RTO rollback ≤ 30 мин, данные целы | 1 раз в квартал |

### 7.2 Правила проведения

- Все chaos-тесты проводятся в staging-окружении, идентичном production
- CE-006 и CE-010 (полный региональный failover) проводятся в non-business hours с предварительным уведомлением stakeholders за 72 часа
- Результаты каждого теста документируются в Post-Mortem (даже при успехе)
- После каждого теста проверяется целостность данных (TBox, ABox, Version Store, audit logs)
- При failure теста: открывается issue, назначается владелец, баг фиксится до следующего раунда
- Автоматизация: CE-001–CE-004 и CE-007 выполняются через GitLab CI (chaos stage) с использованием `chaos-mesh` или `litmus`

### 7.3 Критерии успеха

| Сценарий | Условие успеха |
|----------|----------------|
| CE-001 — CDN | RTO ≤ 5 мин, RPO = 0, статика доступна |
| CE-002 — WAF | RTO ≤ 10 мин, RPO = 0, baseline фильтрация активна |
| CE-003 — Edge | RTO ≤ 15 мин, RPO = 0, rate limiting и JWT-валидация работают |
| CE-004 — API Gateway | RTO ≤ 10 мин, RPO = 0, read-only режим активен |
| CE-005 — Neo4j primary | Read-only mode, алерт P0, нет потери данных |
| CE-006 — Region failover | RTO ≤ 30 мин, RPO ≤ 15 мин, все smoke tests пройдены |
| CE-007 — Partial region loss | Graceful degradation, пользовательский SLO ≥ 99.5% для read |
| CE-008 — Inter-region latency | RPO не превышен, alarms корректны |
| CE-009 — Load test region Б | p95 latency ≤ SLO, error rate < 0.1% |
| CE-010 — Rollback | RTO ≤ 30 мин, данные целы, smoke tests пройдены |

---

## 8. On-Premise и Air-Gapped особенности

### 8.1 On-premise (единый регион)

Для on-premise развёртываний VEDO Core multi-region архитектура заменяется отказоустойчивостью на уровне кластера:

| Компонент | On-premise эквивалент |
|-----------|----------------------|
| CDN | Ingress Controller (nginx / haproxy + локальный кэш) |
| WAF | ModSecurity / Coraza в Ingress Controller |
| Edge-вычисления | Не поддерживается (отказоустойчивость на уровне Ingress) |
| Multi-region | Kubernetes cluster spanning 3+ availability zones внутри одной площадки |
| БД | Patroni для PostgreSQL (HA), Neo4j Causal Cluster |
| DNS | Внутренний DNS (CoreDNS / Bind) |

**Требования к on-premise кластеру:**
- 3+ control plane nodes (etcd cluster)
- 5+ worker nodes, распределённых по 3+ стойкам / availability zones
- Отказ одного датацентра в пределах площадки не должен нарушать кворум etcd
- Локальный container registry (Harbor / Nexus) для offline-развёртывания

### 8.2 Air-gapped

В air-gapped среде failover на внешние компоненты (CDN, WAF, Edge) невозможен. Используется минимальная конфигурация:

- **CDN:** Ingress Controller с локальным кэшированием статики (nginx proxy cache)
- **WAF:** Coraza WAF (open-source, sidecar) в Ingress Controller
- **Edge:** Не поддерживается — rate limiting и JWT-валидация на Ingress / API Gateway
- **Multi-region:** Не поддерживается (физическая изоляция площадки)
- **DNS:** Локальный DNS (CoreDNS / внутренний Windows DNS)

**RTO/RPO в air-gapped режиме:**

| Компонент | RTO | RPO | Примечание |
|-----------|-----|-----|------------|
| Отказ узла Kubernetes | 2 мин | 0 | Автоматический перезапуск pod на здоровом узле |
| Отказ PostgreSQL | 5 мин | ≤ 1 мин | Patroni auto-failover |
| Отказ Neo4j | 10 мин | ≤ 1 мин | Causal cluster auto-failover |
| Отказ всего кластера | 4 часа (RTO critical) | 5 мин (TBox), 15 мин (ABox) | Восстановление из backup (согласно ADR-DES.INFRA.backup-policy-strategy) |

---

## 9. Cost Implications

### 9.1 Warm standby регион Б

| Ресурс | Регион А (активный) | Регион Б (warm standby) | Доп. затраты |
|--------|-------------------|------------------------|--------------|
| Kubernetes worker nodes | 6–10 nodes | 3–5 nodes (минимальный) | 50% от региона А |
| Neo4j | 3 nodes (RW cluster) | 1–2 nodes (read replica) | 50–60% |
| PostgreSQL | 2 nodes (primary + HA) | 1 node (hot standby) | 50% |
| Redis | 3 nodes (cluster) | 0 (разогрев при failover) | 0% (standby) |
| RabbitMQ | 3 nodes (cluster) | 0 (восстановление при failover) | 0% (standby) |
| API Gateway + Backend | 3+ реплики каждого | 1 реплика каждого (дежурный режим) | 30–40% |
| CDN | Основной провайдер | Резервный провайдер (минимальный) | 30–50% |
| S3/MinIO | Активный bucket | Replica bucket | 100% (данные дублируются) |
| Network (cross-region) | — | Replication traffic | Зависит от объёма |
| **Итого (приблизительно)** | **100% (baseline)** | | **50–100% дополнительно** |

### 9.2 Оптимизация затрат

- **deployment-strategy-policy:** Регион Б использует `values-warm-standby.yaml` с минимальными репликами и лимитами. При failover применяется `values-production.yaml` с полным профилем
- **HPA (Horizontal Pod Autoscaler):** В дежурном режиме — 1 pod, при failover — автоскейлинг до production-профиля
- **Spot instances для региона Б:** Допустимы для read-only реплик (Neo4j, PostgreSQL) при условии PDB (Pod Disruption Budget)
- **S3/MinIO:** Lifecycle policy для replica bucket — сразу переводить в Glacier / Archive tier (восстановление за 1–5 мин)

---

## 10. Ответственность

### 10.1 Матрица RACI

| Действие | SRE Lead | DevOps | Security Lead | Product Owner |
|-----------|----------|--------|---------------|---------------|
| Определение RTO/RPO | **R** | C | C | A |
| Настройка CDN/WAF/Edge | C | **R** | C | I |
| Мониторинг edge-компонентов | **R** | **R** | I | I |
| Failover decision | **R** | C | C | A |
| Pre-flight checks | C | **R** | I | I |
| Исполнение failover | **R** | C | I | I |
| Post-failover validation | **R** | **R** | I | I |
| Rollback decision | **R** | C | I | A |
| Chaos engineering планирование | **R** | C | I | I |
| Chaos engineering проведение | I | **R** | I | I |
| Cost optimisation | I | **R** | I | A |
| Документирование runbook | **R** | C | I | I |
| Аудит failover учений | **R** | I | I | I |

*R — ответственный, A — утверждающий, C — консультирующий, I — информируемый*

### 10.2 Описание ролей

| Роль | Обязанности |
|------|-------------|
| **SRE Lead** | Владелец политики отказоустойчивости. Принимает решение о failover, утверждает RTO/RPO, планирует chaos engineering, проводит Post-Mortem после инцидентов и учений |
| **DevOps** | Настройка и поддержка CDN/WAF/Edge/API Gateway конфигураций, написание и поддержка automations, проведение chaos-тестов |
| **Security Lead** | Утверждение WAF bypass-режима, оценка рисков при деградации защиты, аудит алертов безопасности |
| **Product Owner** | Утверждение failover решения как представитель бизнеса, коммуникация с заказчиком, согласование окон rollback |

---

## 11. Runbook References

| Runbook | Описание | Расположение |
|---------|----------|-------------|
| **CDN Failover Runbook** | Процедура переключения на резервный CDN-провайдер | `docs/runbooks/cdn-failover.adoc` |
| **WAF Bypass Runbook** | Процедура переключения WAF в monitor-only режим | `docs/runbooks/waf-bypass.adoc` |
| **Edge Fallback Runbook** | Процедура переключения edge-вычислений на API Gateway | `docs/runbooks/edge-fallback.adoc` |
| **API Gateway Degraded Runbook** | Процедура переключения API Gateway в read-only | `docs/runbooks/gateway-degraded.adoc` |
| **Region Failover Runbook** | Полная процедура failover региона А → региона Б | `docs/runbooks/region-failover.adoc` |
| **Region Rollback Runbook** | Процедура обратного переключения на восстановленный регион А | `docs/runbooks/region-rollback.adoc` |
| **Chaos Engineering Runbook** | Планирование и проведение chaos-тестов | `docs/runbooks/chaos-engineering.adoc` |
| **Post-Failover Validation Runbook** | Проверки после успешного failover | `docs/runbooks/post-failover-validation.adoc` |

---

## 12. Open Questions

- Выбор DNS-провайдера для multi-region routing (Route53 / Cloudflare DNS / Yandex DNS) — требуется уточнение по регионам
- Автоматизация pre-flight checks через `vedo-cli` — deferred до имплементации CLI (см. ADR-DES.INFRA.vedo-cli-admin-boundary)
- Возможность active-active (а не warm standby) для региона Б в будущем — deferred, требует CAP-анализа и решения по conflict resolution
- Интеграция chaos-mesh или litmus в GitLab CI для автоматизации chaos-тестов — deferred до имплементации
- Retention журналов failover-учений — требуется уточнение политики аудита

---

## 13. Ссылки

- [ADR-DES.INFRA.monolith-vs-microservices](#adr-desinframonolith-vs-microservices) — микросервисная архитектура VEDO Core
- [ADR-DES.INFRA.fault-tolerance-strategy](#adr-desinfrafault-tolerance-strategy) — политика circuit breaker и режимов деградации
- [ADR-DES.INFRA.recovery-objectives-mandate](#adr-desinfrarecovery-objectives-mandate) — целевые RTO/RPO по типам данных
- [ADR-DES.PROCESS.deployment-strategy-policy](#adr-desprocessdeployment-strategy-policy) — стратегии развёртывания
- [ADR-DES.INFRA.backup-policy-strategy](#adr-desinfrabackup-policy-strategy) — политика резервного копирования
- [ADR-DES.INFRA.airgap-offline-deployment-strategy](#adr-desinfraairgap-offline-deployment-strategy) — поддержка изолированных сред
- [ADR-DES.INFRA.critical-alerts-strategy](#adr-desinfracritical-alerts-strategy) — модель алертов и severity
- `human/constraints/performance.yaml` — глобальные SLO по latency
- `human/constraints/observability.yaml` — требования к наблюдаемости
- `human/artifacts/stack.md` — технологический стек
