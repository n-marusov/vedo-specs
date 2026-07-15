# ADR-DES.API.deprecation-policy-strategy — Политика deprecation API

**Статус:** ПРЕДЛОЖЕНО
**Дата:** 2026-07-16

## Контекст

VEDO Hub предоставляет API для внешних клиентов: REST API (F6), SPARQL endpoint (F10.5), CYPHER endpoint (F10.5), MCP-сервер (F10.3). API endpoint'ы эволюционируют: добавляются новые версии, изменяются форматы, удаляются устаревшие функции.

Стратегия обратной совместимости уже определена в `ADR-DES.API.backward-compatibility-strategy`. Данная ADR фокусируется на процессе deprecation — как и когда уведомлять клиентов об отключении устаревших endpoint'ов.

**Проблемы:**

1. **Внезапное отключение:** Клиенты теряют доступ к API без предупреждения (Twitter/X API shutdown, 2023)
2. **Отсутствие стандартизации:** Нет единого механизма информирования о deprecation
3. **Разные потребности клиентов:** Enterprise-клиентам нужно больше времени для миграции
4. **Отсутствие обратной связи:** Клиенты не видят, что используют deprecated endpoint'ы

**Существующая база:** Процессы уведомления о завершении услуг уже определены в `ADR-BIZ.PROCESS.vendor-retirement-notice-mandate` и `ADR-BIZ.PROCESS.decommission-notification-mandate`. Данная ADR специализирует эти процессы для API endpoint'ов.

## Требование-источник

- `REQ-NFR.API.deprecation-periods`
- `REQ-FUN.API.sunset-header`
- `REQ-USR.API.deprecation-notification`
- `REQ-FUN.API.migration-guide`
- `REQ-BIZ.API.enterprise-grace-period`
- `REQ-USR.UI.status-page-deprecation`
- `REQ-NFR.DATA.deprecation-audit`
- `REQ-NFR.API.decommission-support`
- `REQ-FUN.API.gradual-rollback`

## Решение

Внедрить **единую политику deprecation API**, основанную на RFC 8594 (Sunset header) и согласованную с существующими процессами VEDO.

### 1. Жизненный цикл deprecation

```
День 0: Объявление
├── Sunset header в ответах (RFC 8594)
├── Email + In-app + Status page + Blog
├── Руководство по миграции опубликовано
│
День 30 (minor) / День 60 (major): Промежуточный этап
├── Email-напоминание
├── Снижение лимитов: 50% от исходного
│
День X-7: Финальное напоминание
├── Email-напоминание
├── Снижение лимитов: 10% от исходного
│
День X: Отключение
├── HTTP 410 Gone (вместо 200/400/500)
├── Обновление статус-страницы
├── Email об отключении
│
После отключения:
├── 0–30 дней: полная поддержка миграции
├── 31–90 дней: консультации по документации
└── > 90 дней: self-service
```

### 2. Классификация изменений

| Тип | Срок | Sunset header | Снижение лимитов |
|-----|------|---------------|-------------------|
| **Minor** (обратно совместимые) | ≥ 30 дней | Да | Нет |
| **Major** (breaking changes) | ≥ 90 дней | Да | Да (50% → 10%) |
| **Security** (критические) | Минимальный | Да | По решению |

Сроки согласованы: 30 дней для minor соответствует периоду охлаждения из `account-closure-retention`; 90 дней для major даёт клиентам квартал на миграцию — стандарт индустрии (GitHub, GitLab, AWS).

### 3. Sunset header (RFC 8594)

```http
HTTP/1.1 200 OK
Sunset: Thu, 15 Oct 2026 00:00:00 GMT
Deprecation: true
Link: <https://docs.vedo.io/api/migration-guide>; rel="deprecation"; type="text/html"
```

Заголовки возвращаются во **всех** ответах endpoint'а (200, 400, 500) с момента объявления до отключения. После отключения — только HTTP 410 Gone.

**Middleware в API Gateway:**

```go
func SunsetHeaderMiddleware() gin.HandlerFunc {
    return func(c *gin.Context) {
        dep := getDeprecation(c.Request.URL.Path)
        if dep == nil || time.Now().After(dep.SunsetDate) {
            c.Next()
            return
        }
        c.Header("Deprecation", "true")
        c.Header("Sunset", dep.SunsetDate.Format(http.TimeFormat))
        if dep.GuideURL != "" {
            c.Header("Link", fmt.Sprintf("<%s>; rel=\"deprecation\"; type=\"text/html\"", dep.GuideURL))
        }
        c.Next()
    }
}
```

### 4. Многоуровневые уведомления (согласовано с vendor-retirement-notice-mandate)

| Канал | Когда | Аудитория |
|-------|------|-----------|
| **Sunset header** | Непрерывно, с дня объявления | Все HTTP-клиенты |
| **Email** | День 0, день X-30, день X-7 | Зарегистрированные клиенты |
| **Status page** | День 0, обновляется при изменениях | Все посетители |
| **Blog** | День 0 | Все посетители |
| **In-app** | День 0 | Все пользователи VEDO |

### 5. Снижение лимитов (gradual rollback)

Стимулирует миграцию, не блокируя клиентов мгновенно:

```yaml
gradual_rollback:
  schedule:
    - days_before: 60    # за 60 дней до отключения
      limit_percent: 50
    - days_before: 7
      limit_percent: 10
```

Enterprise-клиенты с активным grace period исключаются из снижения лимитов.

### 6. Grace period (Enterprise)

Дополнительные 30 дней по запросу. Процесс: заявка в поддержку → оценка командой VEDO → подтверждение → обновление `Sunset` для tenant.

```sql
CREATE TABLE api_deprecation_grace_periods (
    id UUID PRIMARY KEY,
    tenant_id UUID NOT NULL,
    endpoint_name VARCHAR(255) NOT NULL,
    original_sunset_date TIMESTAMPTZ NOT NULL,
    extended_sunset_date TIMESTAMPTZ NOT NULL,
    reason TEXT,
    approved_by UUID,
    created_at TIMESTAMPTZ DEFAULT now()
);
```

### 7. Статус-страница

Раздел «API Lifecycle» с тремя категориями:
- **Active:** текущие endpoint'ы
- **Deprecated:** endpoint'ы в процессе deprecation (с Sunset-датой и альтернативой)
- **Decommissioned:** отключённые endpoint'ы (история)

### 8. Конфигурация

```yaml
api_deprecation:
  periods:
    minor_days: 30
    major_days: 90
    enterprise_grace_days: 30
  sunset_header:
    enabled: true
    include_link: true
  notifications:
    email: true
    in_app: true
    status_page: true
    blog: true
    reminder_days: [30, 7]
  gradual_rollback:
    enabled: true
    schedule: [{days_before: 60, limit: 50}, {days_before: 7, limit: 10}]
  audit:
    retention_days: 365
```

## Рассмотренные альтернативы

| Альтернатива | Причина отклонения |
|--------------|--------------------|
| Немедленное отключение без предупреждения | Нарушение ожиданий клиентов; Twitter/X API shutdown — негативный прецедент |
| Только Sunset header без email | Клиенты могут не заметить заголовок; нужны множественные каналы |
| Единый срок для всех изменений | Разное влияние minor/major; 30 дней для major недостаточно |
| Без grace period для Enterprise | Критические интеграции Enterprise требуют больше времени на миграцию |
| Без снижения лимитов | Нет стимула для миграции; клиенты откладывают до последнего дня |
| Без руководства по миграции | Клиенты не знают, как перейти на новую версию |

## Последствия

**Положительные:**

1. **Прозрачность:** Sunset header (RFC 8594) — индустриальный стандарт, поддерживаемый HTTP-клиентами
2. **Достаточное время:** 30 дней для minor, 90 для major — соответствует GitHub/GitLab/AWS практикам
3. **Согласованность с существующими ADR:** Дополняет `backward-compatibility-strategy` и `vendor-retirement-notice-mandate`
4. **Гибкость:** Grace period для Enterprise, снижение лимитов для стимулирования миграции
5. **Минимальное внедрение:** Middleware в существующем API Gateway, не требует нового сервиса

**Отрицательные:**

1. **Поддержка устаревших endpoint'ов:** 30–90 дней дополнительной поддержки deprecated кода
2. **Сложность коммуникации:** Множественные каналы уведомлений требуют координации

**Меры снижения:**

1. Автоматизация через middleware и конфигурацию (Sunset header добавляется декларативно)
2. Шаблоны email-уведомлений и автоматическая рассылка через существующий SMTP-шлюз

**Риски:**

1. **Игнорирование уведомлений:** Клиенты не замечают предупреждения.
   - **Смягчение:** Три канала (header + email + UI) + напоминания за 30 и 7 дней.
2. **Задержка миграции:** Клиенты откладывают до последнего дня.
   - **Смягчение:** Снижение лимитов за 60 и 7 дней стимулирует раннюю миграцию.

## Связанные ADR

- `ADR-DES.API.backward-compatibility-strategy` — стратегия обратной совместимости (определяет, ЧТО считать breaking change)
- `ADR-BIZ.PROCESS.vendor-retirement-notice-mandate` — общий процесс уведомления о завершении услуг
- `ADR-BIZ.PROCESS.decommission-notification-mandate` — уведомление о деактивации (tenant-level, не API-level)
- `ADR-DES.API.unified-root-endpoint-adoption` — единый корневой endpoint (контекст API-архитектуры)

## Чек-лист реализации

- [ ] Sunset header middleware в API Gateway
- [ ] Конфигурация deprecation (YAML: endpoint → sunset_date)
- [ ] Email-уведомления (шаблоны + автоматическая рассылка)
- [ ] In-app уведомления
- [ ] Статус-страница: раздел API Lifecycle
- [ ] Шаблон руководства по миграции
- [ ] Grace period: API + БД-таблица
- [ ] Gradual rollback: снижение лимитов по расписанию
- [ ] HTTP 410 Gone после отключения
- [ ] Аудит deprecation-событий (365 дней)
- [ ] Документация Admin Guide

---
