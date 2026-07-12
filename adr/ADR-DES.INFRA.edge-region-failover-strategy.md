# ADR-DES.INFRA.edge-region-failover-strategy

**Дата:** 2026-05-16  
**Статус:** Принято

## Контекст

CDN, WAF и edge-вычисления (Workers / Lambda@Edge) являются едиными точками отказа (Single Points of Failure) для публичного веб-трафика VEDO Core. API Gateway — единая точка входа для всех backend-сервисов. Регион размещения — единая точка отказа для всей платформы, если нет кросс-региональной архитектуры.

Индустриальные инциденты подтверждают реальность этой угрозы:
- **Cloudflare (2022):** Сбой CDN 21 июня 2022 года затронул 19 датацентров, продлился 30+ минут. Трафик не мог быть перенаправлен, так как CDN был единственной точкой входа.
- **Fastly (2021):** Глобальный сбой 8 июня 2021 года из-за ошибки конфигурации в одном из 3000+ PoP. Продолжительность ~60 минут. Затронуты Amazon, Reddit, The Guardian, New York Times.
- **AWS us-east-1 (2021):** Сбой в US-EAST-1 в декабре 2021 года продлился ~7 часов. Затронуты тысячи сервисов, полагающихся на один регион. Причина — ошибка в процессе автоматического масштабирования.

VEDO Core имеет базовую отказоустойчивость backend-сервисов (ADR-DES.INFRA.fault-tolerance-strategy), но CDN, WAF, Edge и региональный failover не покрыты отдельной политикой. Каждый из этих компонентов при отказе делает платформу полностью или частично недоступной без формальной процедуры переключения.

## Требование-источник
- Edge & Regional Resilience: CDN/WAF/Edge/Region Failover Specification
- [ADR-DES.INFRA.fault-tolerance-strategy](#adr-desinfrafault-tolerance-strategy)
- [ADR-DES.INFRA.data-residency-region-strategy](#adr-desinfradata-residency-region-strategy)
- [ADR-DES.INFRA.critical-alerts-strategy](#adr-desinfracritical-alerts-strategy)
- [ADR-DES.INFRA.recovery-objectives-mandate](#adr-desinfrarecovery-objectives-mandate)
- [ADR-DES.PROCESS.deployment-strategy-policy](#adr-desprocessdeployment-strategy-policy)

## Решение

Принять многоуровневую стратегию отказоустойчивости edge-компонентов и регионов: автоматический bypass для CDN/WAF/Edge, ручной failover между регионами по схеме active-warm standby, и обязательные chaos engineering тесты.

**Целевые RTO/RPO:**

| Компонент | RTO | RPO | Тип failover | Автоматизация |
|-----------|-----|-----|-------------|---------------|
| CDN | 5 мин | 0 | Автоматический к резервному CDN-провайдеру | Полностью автоматический |
| WAF | 10 мин | 0 | Автоматическое переключение в режим monitor-only | Полностью автоматический |
| Edge-вычисления | 15 мин | 0 | Fallback к региональному API Gateway | Автоматический |
| API Gateway (один инстанс) | 10 мин | 0 | Read-only или переключение на standby | Частично автоматический |
| Регион (полный отказ) | 30 мин | ≤ 15 мин | Ручной failover на регион Б (warm standby) | Manual approval SRE |

**Режимы деградации:**

- **CDN bypass:** Статические ассеты идут напрямую из API Gateway. Все функции доступны с увеличенной задержкой.
- **WAF bypass:** Сервис работает без WAF с P0-алертом. Включена baseline-фильтрация на API Gateway (IP rate limiting, JWT-валидация, базовая санитация ввода).
- **Edge fallback:** Rate limiting и JWT-валидация переносятся на региональный API Gateway. Функции A/B-тестирования и geo-routing недоступны.
- **API Gateway degraded:** Режим read-only. Write-операции отклоняются с понятным сообщением и баннером в UI.
- **Region failover:** Регион Б (warm standby) промотируется до active. Write-операции запрещены до завершения failover.

**Multi-region архитектура:**

Два региона: А (active, read/write) и Б (warm standby, read-only до failover). Репликация данных — асинхронная: Neo4j (RPO ≤ 15 мин), PostgreSQL streaming replication (RPO ≤ 5 мин), Redis, RabbitMQ, S3/MinIO CRR.

Write-операции запрещены в регионе Б до завершения failover — это предотвращает split-brain и исключает конфликты данных при асинхронной репликации.

**Процедура failover (ручная, RTO 30 мин):**

1. **Обнаружение (0–5 мин):** Multiple P0 алерты, SRE подтверждает инцидент, диагностика через `vedo-cli diagnose region --region A`
2. **Decision (5–10 мин):** Проверка по failover decision matrix. Необходимые условия: Neo4j primary недоступен > 3 мин, PostgreSQL primary недоступен > 3 мин, API Gateway error rate > 50% за 5 мин, ≥ 2 из 4 backend-сервисов недоступны > 5 мин
3. **Pre-flight (10–15 мин):** Проверка готовности региона Б (health endpoints, replication lag, DNS, сертификаты, S3/MinIO)
4. **Execution (15–25 мин):** Регион А → read-only, promote PostgreSQL, promote Neo4j, scale up сервисы Б, разогрев Redis, DNS-переключение
5. **Validation (25–30 мин):** Smoke test, health checks, проверка целостности данных, Incident Commander объявляет успех

Failover НЕ выполняется при: единичном сбое CDN (покрывается bypass), WAF error rate spike (покрывается bypass), кратковременной потере связности < 5 мин, плановых maintenance window.

**Процедура rollback:**

Ручная, с отдельным окном ≥ 60 мин. Pre-rollback за 24–48 ч: синхронизация схем, восстановление репликации, разогрев кэша, уведомление stakeholders. Execution: drain региона Б → catch-up репликации → DNS-переключение → smoke test региона А.

**Chaos engineering тесты:**

| ID | Сценарий | Частота |
|----|----------|---------|
| CE-001 | Отказ CDN | 1 раз в месяц |
| CE-002 | Отказ WAF | 1 раз в месяц |
| CE-003 | Отказ Edge Workers | 1 раз в 2 недели |
| CE-004 | Отказ API Gateway | 1 раз в 2 недели |
| CE-005 | Отказ Neo4j primary | 1 раз в квартал |
| CE-006 | Полная потеря региона | 1 раз в квартал |
| CE-010 | Rollback после failover | 1 раз в квартал |

**On-premise / air-gapped:** Для single-region on-premise — отказоустойчивость на уровне кластера Kubernetes (Patroni для PostgreSQL, Causal Cluster для Neo4j). Для air-gapped — внешние CDN/WAF/Edge не используются, RTO зависит от скорости восстановления hardware клиента.

Детальная реализация описана в отдельной технической спецификации `human/artifacts/edge-region-failover.md`.

## Рассмотренные альтернативы

| Альтернатива | Причина отклонения |
|--------------|--------------------|
| **Single region без CDN/WAF/Edge** | Самая дешёвая, но не защищает от отказа региона. Инциденты AWS us-east-1 (2021) показали, что полагаться на один регион неприемлемо для production-систем с SLO 99.9%. Любой hardware-сбой в датацентре делает платформу полностью недоступной |
| **Active-active multi-region (оба региона пишут)** | Требует сложного conflict resolution при асинхронной репликации. Для Neo4j и PostgreSQL нет production-ready кросс-региональной active-active репликации без компромиссов по консистентности. Повышает RPO при конфликтах и создаёт риск расхождения данных. Добавляет существенную сложность в миграции и rollback |
| **Полная автоматизация failover без manual подтверждения** | Слишком рискованно для production. False positive (автоматический failover при кратковременном сбое) может привести к ненужному переключению, потере данных (RPO ≤ 15 мин затирается), простою при откате. Инцидент AWS (2021) показал, что автоматический failover может усугубить ситуацию, если регион Б не готов |
| **Использование только managed service (AWS Global Accelerator, Cloudflare Always-On)** | Привязывает VEDO Core к одному cloud-провайдеру, что противоречит ADR-DES.INFRA.data-residency-region-strategy (поддержка Yandex Cloud, AWS, on-premise). Managed services имеют свои единые точки отказа (Cloudflare 2022, Fastly 2021) |
| **Отказ от WAF/Edge, полагаться только на API Gateway rate limiting** | API Gateway не обеспечивает защиту от OWASP Top 10, SQL-инъекций, XSS, bot-трафика. WAF bypass является временным режимом, а не постоянной архитектурой. Rate limiting на API Gateway — только один из слоёв защиты |

## Последствия

**Положительные последствия:**
- Платформа остаётся доступной при отказе CDN, WAF или Edge-вычислений — каждый имеет автоматический bypass с RTO ≤ 15 мин
- Кросс-региональная архитектура защищает от полной потери датацентра (RTO ≤ 30 мин, RPO ≤ 15 мин)
- Write-запрет в регионе Б до failover исключает split-brain и конфликты репликации
- Chaos engineering тесты доказывают работоспособность процедур до реального инцидента
- Ручной failover с decision matrix снижает риск false positive переключения

**Отрицательные последствия:**
- Warm standby регион Б требует 50–100% дополнительных ресурсов (Kubernetes nodes, БД-реплики, сетевой трафик)
- Асинхронная репликация создаёт окно потери данных до 15 мин при полном отказе региона А
- Ручной failover (manual approval SRE) увеличивает RTO до 30 мин против потенциальных 5–10 мин при полной автоматизации
- Multi-region архитектура усложняет операции миграции, развёртывания и тестирования
- Air-gapped и on-premise развёртывания не получают преимуществ CDN/WAF/Edge и multi-region — их RTO зависит от инфраструктуры клиента

**Меры снижения рисков:**

| Риск | Мера |
|------|------|
| False positive при failover | Failover decision matrix с 6 условиями для включения и 6 для отключения. Pre-flight checks региона Б перед каждым failover |
| Split-brain при асинхронной репликации | Fencing token на уровне БД (Neo4j и PostgreSQL). Write-запрет в регионе Б до failover. После failover регион А изолируется (drain + network fence) |
| Replication lag превышает RPO | Мониторинг replication lag с P0-алертом при Neo4j lag > 15 мин и PostgreSQL lag > 5 мин. Автоматическая эскалация SRE при превышении |
| WAF bypass создаёт окно уязвимости | Baseline-фильтрация на API Gateway (IP rate limiting, JWT-валидация, sanitization). Maximum time in bypass: 10 мин до auto-recovery или ручного вмешательства |
| Chaos-тесты не покрывают реальные сценарии | 10 сценариев с разной периодичностью. Обязательный Post-Mortem после каждого теста. Обновление сценариев на основе реальных инцидентов |
| On-premise клиент не готов к отказу | Документированные требования к HA-кластеру (Patroni, Causal Cluster). Runbook для восстановления |

## Ссылки
- [Edge & Regional Resilience Specification v1.0](edge-region-failover.md) — техническая спецификация реализации
- [ADR-DES.INFRA.fault-tolerance-strategy](#adr-desinfrafault-tolerance-strategy) — политика circuit breaker и режимов деградации backend-сервисов
- [ADR-DES.INFRA.data-residency-region-strategy](#adr-desinfradata-residency-region-strategy) — политика мультирегионального развёртывания
- [ADR-DES.INFRA.critical-alerts-strategy](#adr-desinfracritical-alerts-strategy) — модель severity и эскалации алертов
- [ADR-DES.INFRA.recovery-objectives-mandate](#adr-desinfrarecovery-objectives-mandate) — целевые RTO/RPO по типам данных
- [ADR-DES.PROCESS.deployment-strategy-policy](#adr-desprocessdeployment-strategy-policy) — стратегии развёртывания (blue-green, rolling, canary)
- [ADR-DES.INFRA.airgap-offline-deployment-strategy](#adr-desinfraairgap-offline-deployment-strategy) — поддержка изолированных сред

---
