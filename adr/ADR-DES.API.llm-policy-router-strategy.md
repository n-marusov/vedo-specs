# ADR-DES.API.llm-policy-router-strategy

**Дата:** 2026-07-15
**Статус:** ПРЕДЛОЖЕНО

## Контекст

M2 (AI Assistant & Low-Barrier Onboarding) вводит AI-функции, использующие LLM-провайдеров:
- NL→OWL генерация (F14.1, F14.2)
- NL-режим запросов (F10.1, F10.2)
- AI-подсказки по связям (F14.3)

Доступные LLM-провайдеры зависят от типа развёртывания VEDO:

| Тип развёртывания | Доступные LLM |
|---|---|
| **SaaS** | Внешние (OpenAI, Anthropic и др.) |
| **On-premise** | Локальная (разворачивается клиентом внутри периметра) |

Уровни видимости онтологий (Public / Internal / Private — см. `REQ-FUN.DATA.ontology-visibility-levels`) накладывают дополнительные ограничения на использование LLM в SaaS:

| Уровень | SaaS (по умолчанию) | On-premise |
|---------|---------------------|------------|
| **Public** | Внешние LLM разрешены | Локальная LLM |
| **Internal** | Внешние LLM ЗАБЛОКИРОВАНЫ | Локальная LLM |
| **Private** | Внешние LLM ЗАБЛОКИРОВАНЫ | Локальная LLM |

Для SaaS администратор группы/онтологии может явно разрешить внешние LLM для Internal/Private через настройки (см. `REQ-FUN.UI.external-llm-override`).

Без централизованного механизма роутинга логика политик будет дублироваться в каждом AI-компоненте (NL→OWL, NL-запросы, подсказки), что приведёт к расхождению поведения и усложнит аудит.

## Требование-источник

- `REQ-FUN.API.llm-policy`
- `REQ-FUN.DATA.ontology-visibility-levels`
- `REQ-FUN.UI.external-llm-override`
- `REQ-CON.INFRA.saas-llm-limitations`
- `REQ-CON.INFRA.on-premise-llm-limitations`
- `REQ-USR.UI.external-llm-consent`

## Решение

Внедрить **LLM Policy Router** — централизованный компонент на уровне API Gateway, реализующий всю логику принятия решений о маршрутизации LLM-запросов.

**Архитектурная схема:**

```
┌──────────────────────────────────────────────────────────────┐
│                      API Gateway                             │
│                                                              │
│  ┌────────────────────────────────────────────────────────┐  │
│  │               LLM Policy Router                        │  │
│  │                                                        │  │
│  │  1. Принимает запрос с ontology_id                     │  │
│  │  2. Определяет deployment_type из конфига              │  │
│  │  3. Определяет visibility онтологии (cache TTL 60s)    │  │
│  │  4. Для SaaS:                                          │  │
│  │     ├─ Public    → внешний LLM (с consent)            │  │
│  │     └─ Internal/Private:                               │  │
│  │         ├─ проверяет admin overrides                   │  │
│  │         ├─ разрешено → внешний LLM (с consent)        │  │
│  │         └─ запрещено → 403 LLM_BLOCKED_BY_POLICY      │  │
│  │  5. Для On-premise:                                    │  │
│  │     └─ всегда → локальный LLM (без consent)            │  │
│  │  6. Audit log                                          │  │
│  └────────────────────────────────────────────────────────┘  │
│                            │                                 │
│            ┌───────────────┼───────────────┐                 │
│            ▼               ▼               ▼                 │
│  ┌────────────────────────────────────────────────────────┐  │
│  │              LLM Adapter Layer                         │  │
│  ├─────────────┬─────────────┬───────────────────────────┤  │
│  │   OpenAI    │  Anthropic  │   Local (on-premise only) │  │
│  │   Adapter   │   Adapter   │        Adapter            │  │
│  └─────────────┴─────────────┴───────────────────────────┘  │
└──────────────────────────────────────────────────────────────┘
```

**Алгоритм принятия решения:**

```
function route_llm_request(ontology_id, action):
    deployment_type = config.get("deployment.type")

    if deployment_type == "on-premise":
        if not config.has("llm.providers.local.endpoint"):
            return error("LLM_LOCAL_NOT_CONFIGURED")
        audit_log(provider="local", consent_required=false)
        return route_to("local")

    # SaaS
    visibility = ontology_service.get_visibility(ontology_id)  # cached, TTL 60s

    if visibility == "public":
        check_consent(ontology_id)          # REQ-USR.UI.external-llm-consent
        audit_log(visibility="public", provider=selected_provider, consent=consent_status)
        return route_to(selected_external_provider)

    # Internal or Private
    override = check_admin_overrides(ontology_id)  # ontology → group → tenant
    if not override:
        audit_log(status="blocked_by_policy", reason="internal/private, no admin override")
        return error("LLM_BLOCKED_BY_POLICY",
                     "AI-функции заблокированы для Internal/Private онтологий. "
                     "Администратор может разрешить внешние LLM в настройках.")

    # Admin override granted
    check_consent(ontology_id)              # REQ-USR.UI.external-llm-consent
    audit_log(visibility=visibility, override=true, consent=consent_status)
    return route_to(selected_external_provider)
```

**Конфигурация tenant (SaaS):**

```yaml
deployment:
  type: saas

llm:
  providers:
    openai:
      enabled: true
      model: gpt-4
      endpoint: https://api.openai.com/v1
    anthropic:
      enabled: true
      model: claude-3-opus-20240229
      endpoint: https://api.anthropic.com/v1
    local:
      enabled: false  # локальная LLM не предоставляется в SaaS

  policies:
    public:
      allowed_providers: [openai, anthropic]
      require_consent: true
      default: openai
    internal:
      default_blocked: true
      admin_override_allowed: true
      require_consent: true
      allowed_providers_if_override: [openai, anthropic]
    private:
      default_blocked: true
      admin_override_allowed: true
      require_consent: true
      allowed_providers_if_override: [openai, anthropic]
```

**Конфигурация tenant (On-premise):**

```yaml
deployment:
  type: on-premise

llm:
  providers:
    openai:
      enabled: false
    anthropic:
      enabled: false
    local:
      enabled: true
      endpoint: ${VEDO_LLM_LOCAL_ENDPOINT}
      model: ${VEDO_LLM_LOCAL_MODEL}

  policies:
    public:
      allowed_providers: [local]
      require_consent: false
    internal:
      allowed_providers: [local]
      require_consent: false
    private:
      allowed_providers: [local]
      require_consent: false
```

## Рассмотренные альтернативы

| Альтернатива | Причина отклонения |
|--------------|--------------------|
| Hardcoded логика в каждом AI-сервисе | Дублирование кода, риск расхождения политик, усложнение аудита |
| Client-side (UI) политика | Неприемлемо: нельзя полагаться на клиент для enforcement политик безопасности |
| Раздельные билды SaaS / On-premise | Усложнение поставки, два артефакта вместо одного, усложнение CI/CD |
| Единый провайдер для всех (только OpenAI) | Vendor lock-in, невозможность on-premise (нет локальной LLM) |
| Всегда разрешать внешние LLM с consent (без политик видимости) | Неприемлемо для regulated industries: consent на уровне пользователя недостаточен для Internal/Private онтологий |

## Последствия

**Положительные последствия:**

- **Единая точка enforcement:** все AI-функции проходят через один роутер — невозможно обойти политику.
- **Безопасность по умолчанию:** Internal/Private блокируют внешние LLM без явного административного разрешения.
- **Гибкость:** администратор может сознательно разрешить внешние LLM для конкретных онтологий/групп.
- **On-premise ready:** локальная LLM — first-class citizen, конфигурация унифицирована.
- **Audit-ready:** каждое решение роутера логируется с полным контекстом (visibility, override, consent, provider).

**Отрицательные последствия:**

- **Дополнительный сетевой hop:** каждый AI-запрос проходит через API Gateway → LLM Policy Router → Adapter, увеличивая latency на ~5-10 мс.
- **Зависимость от Ontology Service:** для определения visibility нужен вызов к Ontology Service (смягчается кэшированием, TTL 60 сек).
- **Сложность конфигурации:** администратор должен понимать иерархию настроек (онтология > группа > tenant).
- **Latency при недоступности Ontology Service:** если Ontology Service недоступен, а кэш просрочен, AI-запросы будут отклоняться.

**Меры снижения рисков:**

- Кэширование visibility (TTL 60 сек) с graceful fallback: при недоступности Ontology Service использовать последнее известное значение + increment stale-счётчик.
- Latency роутера мониторится через OpenTelemetry (span `llm_policy_router.decision`) с алертом при p95 > 50 мс.
- Шаблоны конфигурации tenant для типовых сценариев (SaaS-public-only, SaaS-enterprise, on-premise).
- Admin UI с валидацией иерархии настроек и preview-эффекта изменений.

## Связанные ADR

- `ADR-DES.SECURITY.nl-query-opt-in-mandate` — дополняется: диалог согласия отображается только для внешних провайдеров; MCP-сервер и on-premise — без согласия.
- `ADR-DES.SECURITY.gitlab-like-organization-model` — расширяется: роли Owner/Admin получают право управлять настройкой external LLM override.
- `ADR-DES.INFRA.airgap-offline-deployment-strategy` — дополняется: on-premise поставка включает конфигурацию локальной LLM.
- `ADR-DES.API.sparql-query-language-strategy` — NL-режим запросов проходит через LLM Policy Router.
- `ADR-DES.INTEGRATION.mcp-server-query-adoption` — MCP-инструменты проходят через тот же роутер.
- `ADR-DES.INFRA.ai-orchestration-service-strategy` — миграция роутера в ai-orchestration-service (см. Addendum).

## Чек-лист реализации

- [ ] LLM Policy Router реализован в API Gateway
- [ ] Адаптеры для OpenAI, Anthropic, Local (OpenAI-compatible API)
- [ ] Административные настройки override в UI
- [ ] Кэширование visibility в Ontology Service (60s TTL)
- [ ] Audit-логирование всех решений роутера
2. Consent-диалог в UI (REQ-USR.UI.external-llm-consent)
- [ ] Визуальный индикатор политики в AI-интерфейсе (REQ-USR.UI.visibility-indicator)
- [ ] Документация Admin Guide: конфигурация LLM для SaaS и on-premise
- [ ] Документация User Guide: как работают AI-функции в зависимости от уровня видимости
- [ ] Интеграционные тесты: все комбинации deployment × visibility × override
- [ ] Метрики: latency роутера, hit-rate кэша visibility, количество блокировок по политике

---

## Addendum: Migration to ai-orchestration-service (2026-07-16)

Per `ADR-DES.INFRA.ai-orchestration-service-strategy`, the LLM Policy Router will be migrated from API Gateway to the dedicated `ai-orchestration-service` in M3.

**M2 (current):** LLM Policy Router operates as API Gateway middleware, as described in this ADR. The architecture diagram above is valid for the M2 prototype phase.

**M3 (planned):** Router moves to `ai-orchestration-service` (Go), becoming gRPC middleware within the AI orchestration service. All LLM-policy decisions happen inside the AI service, not the Gateway. API Gateway proxies AI requests to `ai-orchestration-service` without policy logic.

**What changes:**
- Deployment location: API Gateway -> ai-orchestration-service
- API surface: direct Go function call -> gRPC call from Gateway to ai-orchestration-service
- Resilience requirements from `REQ-NFR.INFRA.router-resilience` apply to ai-orchestration-service (not Gateway)

**What stays the same:**
- Policy decision algorithm (deployment type x visibility x admin override)
- Configuration schema (tenant YAML)
- Audit logging format
- Caching strategy (visibility TTL 60s)

**Related:** `ADR-DES.INFRA.ai-orchestration-service-strategy` — full rationale for the extraction.
