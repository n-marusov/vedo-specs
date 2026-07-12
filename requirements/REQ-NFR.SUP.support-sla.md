# SLA Поддержки

| Атрибут | Значение |
|---------|----------|
| **ID** | REQ-NFR.SUP.support-sla |
| **Уровень** | NFR |
| **Атрибут качества** | Supportability |
| **Приоритет** | P0 |
| **Статус** | УТВЕРЖДЕНО |
| **Источник** | — |
| **Критерии приёмки** | — |

---

## Назначение

Фиксирует уровни поддержки VEDO Core для SaaS, on-premise и enterprise deployments, включая модель критичности, время реакции, целевые workaround/TTR, каналы поддержки и исключения из SLA.

## Классификация инцидентов

| Уровень | Описание | Примеры |
|---------|----------|---------|
| P0 (Critical) | Полная потеря сервиса, потеря данных, серьезная уязвимость безопасности | Neo4j cluster недоступен, утеря данных коммитов, возможность выполнения произвольного кода |
| P1 (High) | Значительная деградация, часть пользователей не работает, критическая функция недоступна | SPARQL endpoint недоступен, reasoner не запускается, GraphQL для навигации недоступен |
| P2 (Medium) | Некритичная функциональность нарушена, работа возможна с обходными путями | Ошибка визуализации графа, медленная загрузка интерфейса |
| P3 (Low) | Косметические проблемы, ошибки в документации, вопросы по использованию | Неправильный шрифт, опечатка, уточняющий вопрос |

## Уровни поддержки

| Тариф / deployment | P0 response | P1 response | P2/P3 | Каналы |
|--------------------|-------------|-------------|-------|--------|
| SaaS MVP / Community | Best effort | Best effort | Best effort | GitHub Issues, Community Forum |
| SaaS Professional | <= 2 часа | <= 8 часов | Best effort, target 2 дня | Web tickets, Email |
| SaaS Business | <= 1 час | <= 4 часа | <= 2 дня | Web tickets, Email, Chat |
| On-premise Community | Best effort | Best effort | Best effort | GitHub Issues |
| On-premise Enterprise | <= 2 часа реакция, <= 8 часов workaround | <= 4 часа реакция | <= 2 дня | Dedicated chat, Email, Support portal |
| Enterprise Custom | <= 30 минут response, <= 2 часа TTR | <= 2 часа response, <= 4 часа TTR | <= 1 день | 24/7 phone, support portal, dedicated chat, customer success manager |

Best effort означает, что команда VEDO старается ответить и решить проблему в разумные сроки, обычно 1-2 дня для P0/P1, но без юридических обязательств, SLA credits или штрафов.

## Формальный Enterprise SLA

| Параметр | P0 | P1 | P2 | P3 |
|----------|----|----|----|----|
| Response Time | <= 30 минут | <= 2 часа | <= 8 часов | <= 2 рабочих дня |
| Time-to-Diagnose (TTD) | <= 10 минут | <= 30 минут | <= 2 часа | Не применяется |
| Time-to-Restore (TTR) | <= 2 часа | <= 4 часа | <= 3 дня | <= 10 дней |
| Availability SLO | 99.95% Enterprise, 99.9% Professional | Не применяется | Не применяется | Не применяется |

Response Time - первый ответ поддержки, не обязательно полное решение. TTR - восстановление сервиса или предоставление согласованного workaround. Resolution - полное устранение причины инцидента.

## Каналы и рабочее время

- P0/P1 для Enterprise Custom обслуживаются 24/7.
- P0 для Enterprise должен иметь телефонный канал с обязательным подтверждением в ticket.
- P1/P2/P3 обрабатываются через support portal, email или dedicated chat.
- P2/P3 могут обслуживаться в рабочее время 5/2, например понедельник-пятница 09:00-18:00 по МСК, если contract не требует 24/7.
- Закрытие инцидента подтверждается мониторингом health endpoint, восстановлением affected workflow или письменным подтверждением customer.

## Исключения из SLA

| Ситуация | Действие VEDO Core | Обоснование |
|----------|--------------------|-------------|
| Плановое обслуживание | Уведомление за 7-14 дней | Не считается простоем при соблюдении maintenance policy |
| Форс-мажор | SLA timer останавливается | Пожар, наводнение, отключение электричества, недоступность провайдера вне контроля VEDO |
| Ошибки заказчика в on-premise | Консультации и best effort помощь | Неправильная сеть, нехватка диска, неподдерживаемая конфигурация |
| Feature request | Обработка через backlog | Не является incident SLA |
| Customer-managed observability replacement | Customer responsibility, если не входит в contract | VEDO не контролирует external tooling заказчика |

## Компенсации SLA

SLA credits применяются только для платных SaaS Business, On-premise Enterprise и Enterprise Custom contracts, если они явно включены в договор. Рекомендуемый диапазон: 5-10% месячного платежа за каждый час превышения P0/P1 SLA, с monthly cap, определенным в contract.

## YAML-Сводка

```yaml
support_sla:
  saas_mvp:
    level: best_effort
    channels: [github_issues, community_forum]
    response_time: no_guarantee
  saas_professional:
    p0_response: 2_hours
    p1_response: 8_hours
    p2_p3_target: 2_days_best_effort
    channels: [web_ticket, email]
  saas_business:
    p0_response: 1_hour
    p1_response: 4_hours
    p2_p3_response: 2_days
    availability_slo: 99.9_percent
    channels: [web_ticket, email, chat]
  on_premise_enterprise:
    p0_response: 2_hours
    p0_workaround: 8_hours
    p1_response: 4_hours
    channels: [dedicated_chat, email, portal]
  enterprise_custom:
    p0_response: 30_minutes
    p0_ttd: 10_minutes
    p0_ttr: 2_hours
    p1_response: 2_hours
    p1_ttd: 30_minutes
    p1_ttr: 4_hours
    availability_slo: 99.95_percent
    support_24_7: true
    sla_credits: true
```

## Формулировка для заказчика

В рамках Enterprise SLA VEDO гарантирует время реакции на P0 инцидент 30 минут, на P1 - 2 часа, круглосуточно. Целевое время восстановления: P0 - 2 часа, P1 - 4 часа. Availability SLO для Enterprise - 99.95%, для SaaS Business - 99.9%. Каналы: телефон, ticket system, dedicated chat и customer success manager. SLA credits применяются только если они явно включены в договор.

## Бизнес-правила

- SaaS MVP и On-premise Community используют best effort support без юридических SLA guarantees.
- SaaS Professional target: P0 response <= 2 часа, P1 response <= 8 часов.
- SaaS Business target: P0 response <= 1 час, P1 response <= 4 часа, P2/P3 <= 2 дня.
- On-premise Enterprise target: реакция на P0 <= 2 часа, workaround для P0 <= 8 часов, реакция на P1 <= 4 часа.
- Enterprise Custom target: P0 response <= 30 минут, P0 TTR <= 2 часа, P1 response <= 2 часа, P1 TTR <= 4 часа.
- Enterprise TTD targets: P0 <= 10 минут, P1 <= 30 минут.
- Enterprise availability SLO: 99.95%; SaaS Business/Professional baseline: 99.9% where contract includes availability commitment.
- P0/P1 Enterprise support must be available 24/7.
- SLA credits are contractual and not enabled by default.
- Planned maintenance with required notice does not count as downtime.
- Customer-caused on-premise incidents receive support, but TTR is not guaranteed unless contract explicitly includes managed operations.

## Break-glass доступ (Emergency Admin)

### Уровни

| Уровень | Механизм | Зависимость от IdP |
|---------|----------|-------------------|
| **L1: Emergency Admin** | Отдельная учётная запись вне Keycloak, пароль (bcrypt/argon2), разделённый через Shamir's Secret Sharing (M из N), хранится в сейфе | Нет |
| **L2: Recovery Key** | Криптографический ключ (PEM/PGP) для подписи временного JWT, доступ через `vedo-cli`, ключ в offline-сейфе | Нет |
| **L3: Physical Console** | Локальный Unix-пользователь на сервере (on-premise) или `kubectl exec` (SaaS managed) | Нет |

### Поведение

- **Активация:** `/emergency/login` — endpoint не зависит от Keycloak.
- **Функции Emergency Admin:** `vedo-cli emergency readonly`, `vedo-cli restore`, `vedo-cli diagnose`, управление учётными записями администраторов.
- **Аудит:** каждая активация → оповещение всех держателей частей пароля и security-команды; immutable audit log.
- **Деактивация:** после восстановления IdP пароль принудительно сбрасывается и заново разделяется.
- **Ротация:** каждые 90 дней и после каждого использования.

### Обоснование

Урок Microsoft 365 outage (2026): длительный простой из-за недоступности внешнего IdP. Emergency Admin вне Keycloak — обязательный fallback. Физический доступ (L3) не заменяет L1, так как может быть недоступен в облачных средах.

### Что не допускается

- L3 (физическая консоль) как единственный break-glass.
- Хранение пароля Emergency Admin в том же Keycloak.
- Отсутствие аудита/оповещений при активации.
- Отсутствие ротации (пароль без срока действия).

### Закрывает

- `5.3 Тех. обеспеченность ЖЦ × Эксплуатация / Обслуживание`

## Vendor retirement notice period

Нормативные сроки, каналы и доказательства уведомлений при decommission/migration дополнительно фиксируются в `decommission-notification-policy.md`.

### Срок уведомления

Минимальный срок уведомления о прекращении сервиса: **180 дней** от даты персонального уведомления конкретного заказчика.

| Параметр | Значение |
|----------|----------|
| Точка отсчёта | Дата персонального уведомления администратора заказчика (email, зарегистрированный в системе) |
| Публичное объявление | Может быть раньше, но 180 дней отсчитываются от персонального уведомления |
| Подтверждение получения | ≤ 7 дней после отправки; при отсутствии — дублирование через альтернативные каналы |

### Фазы 180-дневного периода

| Фаза | Срок | Действия |
|------|------|----------|
| Уведомление | День 0 | Персональное уведомление + статус-панель |
| Планирование | Дни 0–30 | Согласование плана миграции, бюджета; VEDO предоставляет расширенную поддержку |
| Подготовка | Дни 30–90 | Decommission drill, обучение персонала |
| Миграция | Дни 90–150 | Полный decommission export, проверка целостности, импорт в целевую систему |
| Завершение | Дни 150–180 | Подтверждение миграции; финальный purge; сертификат уничтожения |
| Пост-прекращение | День 180+ | Данные удалены; экспортный пакет доступен ещё 30 дней |

### Обязанности VEDO

- Сервис полностью функционален весь 180-дневный период.
- Расширенная поддержка: приоритетные тикеты по миграции.
- Бесплатный decommission export (все связанные операции не тарифицируются).
- Инженер VEDO для консультаций (до 10 часов).
- Срок 180 дней зафиксирован в контракте/ToS.

### Исключения

| Ситуация | Срок | Обоснование |
|----------|------|-------------|
| Форс-мажор (банкротство, суд) | ≥ 90 дней (насколько возможно) | Обстоятельства непреодолимой силы |
| Нарушение ToS (нелегальный контент, атаки) | 30 дней | Предотвращение ущерба третьим лицам |
| Бесплатный trial/community | 30 дней | Не enterprise-уровень |

### Что не допускается

- Отсчёт от публичного объявления без персонального уведомления.
- Сокращение срока для enterprise без форс-мажора.
- Отключение экспорта до истечения 180 дней.
- Удаление данных без письменного подтверждения заказчика.

### Обоснование

Урок vendor shutdown (2026): 90 дней недостаточно для enterprise — бюджетный цикл, миграция, обучение, compliance. 180 дней — индустриальный стандарт (AWS, Google Cloud, Azure).

### Закрывает

- `5.4 Тех. обеспеченность ЖЦ × Утилизация`

## Open questions

- Нет открытых вопросов по support SLA.
