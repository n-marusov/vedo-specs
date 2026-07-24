# BOLA/BFLA Negative Authorization Tests Specification v1.0

| Атрибут | Значение |
|---------|----------|
| **ID** | REQ-NFR.SECURITY.bola-bfla-negative-tests |
| **Уровень** | NFR |
| **Атрибут качества** | Functionality |
| **Приоритет** | P0 |
| **Статус** | УТВЕРЖДЕНО |
| **Источник** | — |
| **Критерии приёмки** | — |

---

> **Domain:** Security
> **Status:** Approved
> **Applies to:** API Gateway, Ontology Service, Auth Service, Collaboration Service, all backend components
> **OWASP References:** ASVS 4.0.3 V4 (Access Control), V5 (Authorization), API Security Top 10 #1 (BOLA)

---

## 1. Definition

### 1.1 BOLA (Broken Object Level Authorization)

BOLA возникает, когда API не проверяет, имеет ли аутентифицированный пользователь право доступа к конкретному объекту по его ID. Атакующий подменяет идентификатор объекта (например, `tenantId`, `ontologyId`) в запросе и получает доступ к чужому объекту.

### 1.2 BFLA (Broken Function Level Authorization)

BFLA возникает, когда пользователь с недостаточными привилегиями вызывает функцию (эндпоинт), требующую более высокой роли. Подмена `role` или вызов admin-эндпоинта рядовым пользователем.

### 1.3 IDOR (Insecure Direct Object Reference)

Частный случай BOLA, когда объектный ID является предсказуемым (integer, base64-encoded UUID, последовательный идентификатор), что упрощает атаку перебором.

### 1.4 Контекст VEDO Core

VEDO Core — мультитенантная платформа управления онтологиями. Каждый запрос содержит:
- `tenantId` в URL пути или JWT
- Идентификаторы онтологий, классов, свойств, индивидуумов
- Роль пользователя в контексте tenant

Авторизация проверяется на уровне API Gateway (JWT + tenant context) и на уровне сервисов (object-level policy).

---

## 2. Перечень endpoint-классов с обязательными negative тестами

### 2.1 REST API

| № | Endpoint класс | Объектные ID | Приоритет |
|---|---------------|--------------|-----------|
| 1 | `/api/v1/tenants/{tenantId}/*` | `tenantId` | P0 |
| 2 | `/api/v1/ontologies/{ontologyId}/*` | `ontologyId`, `tenantId` | P0 |
| 3 | `/api/v1/branches/{branchId}/*` | `branchId`, `ontologyId` | P0 |
| 4 | `/api/v1/commits/{commitId}/*` | `commitId`, `ontologyId`, `branchId` | P0 |
| 5 | `/api/v1/users/{userId}/*` (admin) | `userId` | P0 |
| 6 | `/api/v1/roles/*` (admin) | `roleId`, `tenantId` | P0 |
| 7 | `/api/v1/invites/{inviteId}` | `inviteId` | P1 |
| 8 | `/api/v1/backups/*` (admin) | `backupId`, `tenantId` | P0 |

### 2.2 GraphQL API

| № | Endpoint класс | Объектные ID | Приоритет |
|---|---------------|--------------|-----------|
| 9 | GraphQL мутации (ontologyId в args) | `ontologyId`, `classId`, `individualId`, `propertyId` | P0 |
| 10 | GraphQL запросы (ontologyId в args) | `ontologyId` | P0 |

### 2.3 SPARQL

| № | Endpoint класс | Объектные ID | Приоритет |
|---|---------------|--------------|-----------|
| 11 | `POST /api/v1/sparql` (с `default-graph-uri`) | `ontologyId` | P1 |

### 2.4 WebSocket / Collaboration

| № | Endpoint класс | Объектные ID | Приоритет |
|---|---------------|--------------|-----------|
| 12 | WebSocket `JoinRoom` (CollaborationService) | `ontologyId` | P0 |

### 2.5 gRPC (internal)

| № | Endpoint класс | Объектные ID | Приоритет |
|---|---------------|--------------|-----------|
| 13 | Internal gRPC вызовы между сервисами | `tenantId`, `ontologyId` | P1 |

---

## 3. Классификация тестов по типам

### Тип A — Cross-tenant BOLA

Пользователь Tenant A пытается получить доступ к объекту Tenant B.

```yaml
test_case:
  name: "Cross-tenant ontology GET — Tenant A requests Tenant B's ontology"
  type: BOLA_CROSS_TENANT
  endpoint: /api/v1/ontologies/{ontologyId}
  method: GET
  object_ids:
    tenantId: tenant_B_id
    ontologyId: ontology_B_id
  auth_user: user_tenant_A
  expected_status: 403
  expected_error_code: FORBIDDEN_CROSS_TENANT_ACCESS
  audit_log_must_contain: true
```

### Тип B — Cross-object BOLA (внутри tenant)

Пользователь Tenant A пытается получить доступ к объекту Tenant A, на который у него нет прав.

```yaml
test_case:
  name: "Cross-object ontology GET — user without role reads ontology"
  type: BOLA_CROSS_OBJECT
  endpoint: /api/v1/ontologies/{ontologyId}
  method: GET
  object_ids:
    tenantId: tenant_A_id
    ontologyId: ontology_restricted_A
  auth_user: user_tenant_A_no_role
  expected_status: 403
  expected_error_code: FORBIDDEN_INSUFFICIENT_ROLE
  audit_log_must_contain: true
```

### Тип C — BFLA (Privilege escalation)

Пользователь с ролью ниже необходимой вызывает эндпоинт.

```yaml
test_case:
  name: "BFLA — viewer calls ontology DELETE"
  type: BFLA
  endpoint: /api/v1/ontologies/{ontologyId}
  method: DELETE
  object_ids:
    tenantId: tenant_A_id
    ontologyId: ontology_A_id
  auth_user: user_viewer
  expected_status: 403
  expected_error_code: FORBIDDEN_INSUFFICIENT_ROLE
  audit_log_must_contain: true
```

### Тип D — IDOR через guessable IDs

Подстановка предсказуемых ID (numeric+1, UUID variant, base64 decoded).

```yaml
test_case:
  name: "IDOR guessable — sequential numeric ID probe"
  type: IDOR_GUESSABLE
  endpoint: /api/v1/ontologies/{ontologyId}
  method: GET
  object_ids:
    tenantId: tenant_A_id
    ontologyId: "00000000-0000-0000-0000-000000000101"
  auth_user: user_tenant_A
  expected_status: 403
  expected_error_code: FORBIDDEN_OBJECT_NOT_FOUND_OR_ACCESS_DENIED
  audit_log_must_contain: true
```

> **Важно:** Ответ должен быть `403`, НЕ `404`. `404` раскрывает атакующему факт существования объекта. Всегда используйте универсальный ответ: `403 Forbidden` с кодом `FORBIDDEN_OBJECT_NOT_FOUND_OR_ACCESS_DENIED` для неизвестных/чужих объектов.

---

## 4. Критерии PASS/FAIL

### 4.1 Обязательные (gate-breaking)

| Критерий | Значение | Нарушение блокирует |
|----------|----------|---------------------|
| P0 negative тесты возвращают HTTP 403 | 100% тестов | Merge Request, Release |
| P1 negative тесты возвращают HTTP 403 | ≥ 90% тестов | Release |
| Тесты возвращают 404 | 0% тестов | Merge Request |
| Тесты возвращают 500 | 0% тестов | Merge Request |
| audit_log включён | 100% тестов | Release |

### 4.2 Рекомендованные (non-blocking, метрики)

| Метрика | Цель |
|---------|------|
| SAST-покрытие эндпоинтов авторизационной проверкой | 100% |
| Fuzzing-покрытие IDOR-уязвимых эндпоинтов | ≥ 80% |
| DAST-покрытие в staging | ≥ 60% |

### 4.3 Формальное правило

```
∀ t ∈ Tests(P0) : t.expected_status == 403 ∧ t.actual_status == 403
∀ t ∈ Tests(P0) : t.actual_status ∉ {404, 500}
∀ t ∈ Tests(P0) : t.audit_log_must_contain == true → ∃ audit_record
```

---

## 5. YAML-шаблон теста

### 5.1 Полный шаблон

```yaml
# BOLA/BFLA Negative Test — шаблон
# Обязательные поля: name, type, endpoint, method, object_ids,
#   auth_user, expected_status, expected_error_code
# Опциональные: description, setup, cleanup, fuzz_enabled, tags

test_case:
  # Уникальное имя теста (pattern: "{type}_{endpoint_class}_{scenario}")
  name: "BOLA_CROSS_TENANT_ontologies_GET_tenantB"

  # Тип: BOLA_CROSS_TENANT | BOLA_CROSS_OBJECT | BFLA | IDOR_GUESSABLE
  type: BOLA_CROSS_TENANT

  # Endpoint (c path parameters в нотации {param})
  endpoint: "/api/v1/ontologies/{ontologyId}"

  # HTTP метод
  method: GET

  # Значения path/query/body параметров для подстановки
  object_ids:
    tenantId: "tenant_B_id"        # tenant, к которому НЕТ доступа
    ontologyId: "ontology_B_id"    # объект, к которому НЕТ доступа

  # Пользователь, от имени которого выполняется запрос
  # Должен существовать в тестовом окружении и иметь валидный JWT
  auth_user: "user_tenant_A_only"

  # Ожидаемый HTTP статус
  expected_status: 403

  # Ожидаемый error code из ответа (поле code в JSON-ответе)
  expected_error_code: "FORBIDDEN_CROSS_TENANT_ACCESS"

  # Требуется ли проверка audit log
  audit_log_must_contain: true

  # Опциональные поля
  description: "Пользователь tenant_A пытается прочитать онтологию tenant_B"
  setup:
    create_ontology: true
    create_users: ["user_tenant_A_only", "user_tenant_B"]
  cleanup:
    delete_ontology: true
  fuzz_enabled: true
  tags:
    - P0
    - security
    - authorization
    - regression
```

### 5.2 Матрица expected_error_code

| Тип | expected_error_code | Когда |
|-----|-------------------|-------|
| BOLA_CROSS_TENANT | `FORBIDDEN_CROSS_TENANT_ACCESS` | TenantId mismatch |
| BOLA_CROSS_OBJECT | `FORBIDDEN_INSUFFICIENT_ROLE` | Нет роли на объект |
| BFLA | `FORBIDDEN_INSUFFICIENT_ROLE` | Недостаточно прав для операции |
| BFLA (admin) | `FORBIDDEN_ADMIN_ONLY` | Эндпоинт только для админов |
| IDOR_GUESSABLE | `FORBIDDEN_OBJECT_NOT_FOUND_OR_ACCESS_DENIED` | Объект не найден или нет доступа (универсальный) |

---

## 6. Требования к CI-автоматизации

### 6.1 Этапы в GitLab CI

```mermaid
flowchart TD
    A[Merge Request Created] --> B{Изменяет API?}
    B -->|No| C[Skip BOLA tests]
    B -->|Yes| D[Stage: SAST Scan]
    D --> E{SAST нашёл\nнеавторизованный endpoint?}
    E -->|Yes| F[FAIL: MR blocked]
    E -->|No| G[Stage: Build Test Containers]
    G --> H[Stage: Deploy Test Environment]
    H --> I[Stage: Contract Tests\n(API Gateway)]
    I --> J[Stage: BOLA/BFLA Negative Tests]
    J --> K{Все P0 тесты\nпрошли?}
    K -->|No| L[FAIL: Report to MR]
    K -->|Yes| M[Stage: Fuzzing (IDOR)]
    M --> N{Fuzzing нашёл\nуязвимости?}
    N -->|Yes| O[WARN: Create Security Issue]
    N -->|No| P[Stage: DAST (Staging)]
    P --> Q[PASS: Merge Ready]
    L --> R[Create blocking review]
    R --> B
    O --> P
```

### 6.2 Job definitions

```yaml
# .gitlab-ci.yml — BOLA/BFLA тестирование

bola-negative-tests:
  stage: security
  needs: ["deploy-test-env"]
  script:
    - cd tests/security/bola
    - python -m pytest bola_negative_tests.py --junitxml=report.xml --tb=short
    - python check_coverage.py --p0-required=100 --p1-required=90
  artifacts:
    reports:
      junit: tests/security/bola/report.xml
    paths:
      - tests/security/bola/report.xml
  rules:
    - if: '$CI_PIPELINE_SOURCE == "merge_request_event"'
      changes:
        - "llm/src/api-gateway/**/*"
        - "llm/src/auth-service/**/*"
        - "llm/src/ontology-service/**/*"
        - "tests/security/**/*"

bola-fuzzing:
  stage: security
  needs: ["deploy-test-env"]
  script:
    - cd tests/security/fuzzing
    - python fuzz_idor.py --target="$TEST_URL" --duration=300
  allow_failure: true
  artifacts:
    paths:
      - tests/security/fuzzing/report.json

sast-bola-check:
  stage: security
  script:
    - python scripts/check-authz-coverage.py \
        --api-spec=docs/api/openapi.yaml \
        --sources=llm/src/ \
        --exclude-patterns=health,oauth,browse
  rules:
    - if: '$CI_PIPELINE_SOURCE == "merge_request_event"'
```

### 6.3 Требования к инфраструктуре тестов

- Изолированный test tenant с предустановленными данными (min 2 tenant, min 3 пользователя с разными ролями)
- Возможность генерации JWT для любого тестового пользователя
- Возможность сброса состояния между тестами (reset test data fixture)
- Audit log должен быть доступен для чтения в тестовом окружении

---

## 7. Требования к Runtime Detection (Production)

### 7.1 Автоматические детекторы

| Сигнатура | Условие срабатывания | Действие | Приоритет |
|-----------|---------------------|----------|-----------|
| High 403 rate | Доля 403 > 5% на эндпоинт за 5 мин | P2 алерт, дашборд | P2 |
| Sequential 403 sweep | Последовательные 403 от одного IP на разные `tenantId` > 10 за 1 минуту | Временная блокировка IP на 15 мин + P1 алерт | P1 |
| WAF: tenant mismatch | `tenantId` в JWT не совпадает с `tenantId` в URL path (если детектируемо) | Блокировка запроса + P0 алерт | P0 |
| Anomalous object ID pattern | Последовательные запросы с инкрементальными ID (1, 2, 3...) или UUIDv1 timestamp | P2 алерт + rate-limit усиление | P2 |

### 7.2 Метрики для дашборда (Grafana)

```
bola_403_total{endpoint, tenant, user}         — счётчик 403 по эндпоинтам
bola_403_rate{endpoint}                         — доля 403 от общего числа запросов
bola_blocked_ips                                 — количество временно заблокированных IP
bola_fuzzing_detected                            — количество детектированных IDOR-попыток
bola_test_coverage_p0                            — % P0 эндпоинтов с negative тестами
bola_test_coverage_p1                            — % P1 эндпоинтов с negative тестами
```

### 7.3 Логирование

Каждый 403, вызванный BOLA/BFLA-проверкой, ДОЛЖЕН содержать:
```json
{
  "event": "authorization.denied",
  "reason": "FORBIDDEN_CROSS_TENANT_ACCESS",
  "user_id": "uuid",
  "tenant_id_requested": "tenant_B",
  "tenant_id_actual": "tenant_A", 
  "object_type": "ontology",
  "object_id": "ontology_B_id",
  "source_ip": "10.0.0.1",
  "user_agent": "Mozilla/..."
}
```

---

## 8. Список исключений

| Endpoint | Причина исключения |
|----------|-------------------|
| `GET /api/v1/public/browse/*` | Публичный read-only API, без авторизации |
| `GET /api/v1/health` | Нет объектных ID, internal endpoint |
| `GET /api/v1/ready` | Нет объектных ID, internal endpoint |
| `POST /api/v1/auth/*` | Pre-authentication, нет tenant context |
| `POST /api/v1/oauth/*` | Pre-authentication, нет tenant context |
| `GET /api/v1/invites/{inviteId}?code={code}` | Invite code — одноразовый токен, авторизация через code, не через JWT |
| `GET /openapi.json` | Публичная OpenAPI spec |
| `GET /graphql` (introspection) | Публичная introspection (если включена) |

---

## 9. Ответственность

| Роль | Обязанности |
|------|-------------|
| **Security Architect** | Определение политики, типов тестов, критериев PASS/FAIL, аудит coverage |
| **Backend Developer** | Написание negative тестов для своих эндпоинтов (по шаблону), имплементация middleware |
| **QA Engineer** | Поддержка тестовой инфраструктуры, прогон fuzzing/DAST, верификация результатов |
| **DevOps** | Интеграция BOLA-тестов в CI, настройка runtime detection, дашборды |
| **Security Reviewer** | Проверка MR на BOLA (чек-лист из раздела 13) |

---

## 10. Compliance: OWASP ASVS

### 10.1 ASVS V4 — Access Control Verification

| ASVS ID | Требование | Покрывается |
|---------|-----------|-------------|
| V4.1.1 | "Пользователь может получить доступ только к тем объектам, на которые у него есть права" | Типы A, B, D |
| V4.1.2 | "API проверяет ownership на tenant level" | Тип A |
| V4.1.3 | "Доступ к объектам по ID требует проверки прав" | Все типы |
| V4.2.1 | "Защита от IDOR через непредсказуемые ID" | Тип D |
| V4.2.2 | "Ответ 403, а не 404 для скрытия существования объекта" | Все типы |

### 10.2 ASVS V5 — Authorization Verification

| ASVS ID | Требование | Покрывается |
|---------|-----------|-------------|
| V5.1.1 | "Чёткое разделение ролей и прав" | Тип C (BFLA) |
| V5.1.2 | "Проверка прав на каждый запрос" | Типы A, B, C, D |
| V5.2.1 | "Отсутствие эскалации привилегий" | Тип C |
| V5.2.3 | "Проверка авторизации на function level" | Тип C |

### 10.3 OWASP API Security Top 10

| API2023 | Категория | Связь |
|---------|-----------|-------|
| API1:2023 | Broken Object Level Authorization | Полное покрытие (все 4 типа) |
| API2:2023 | Broken Authentication | Не в scope (отдельный артефакт) |
| API5:2023 | Broken Function Level Authorization | Тип C (BFLA) |
| API7:2023 | Server Side Request Forgery | Не в scope |

---

## 11. How to write a BOLA test — пошаговая инструкция

### Шаг 1: Определите endpoint

```
Endpoint: DELETE /api/v1/ontologies/{ontologyId}
Method: DELETE
Object IDs: ontologyId, tenantId (из JWT)
```

### Шаг 2: Определите типы negative-сценариев

Для каждого объектного ID в запросе напишите минимум 1 тест каждого применимого типа:

| Тип | Сценарий |
|-----|---------|
| A | Другой tenant |
| B | Другой пользователь, без прав на объект |
| C | Недостаточная роль |
| D | Предсказуемый ID |

### Шаг 3: Определите тестовых пользователей

```yaml
users:
  alice_tenant_A: { tenant: tenant_A, role: admin }
  bob_tenant_A:   { tenant: tenant_A, role: viewer }
  eve_tenant_B:   { tenant: tenant_B, role: editor }
```

### Шаг 4: Напишите тест (YAML + код)

Сначала YAML-описание (документация и CI-анализ):

```yaml
test_case:
  name: "BFLA_ontologies_DELETE_viewer"
  type: BFLA
  endpoint: "/api/v1/ontologies/{ontologyId}"
  method: DELETE
  object_ids:
    tenantId: tenant_A_id
    ontologyId: ontology_A_id
  auth_user: bob_tenant_A
  expected_status: 403
  expected_error_code: "FORBIDDEN_INSUFFICIENT_ROLE"
  audit_log_must_contain: true
```

Затем код теста.

### Шаг 5: Запустите локально

```bash
# Запуск всех BOLA тестов
pytest tests/security/bola/ -v

# Запуск конкретного теста
pytest tests/security/bola/test_ontologies.py::test_bfla_viewer_delete_ontology -v

# Проверка coverage
python tests/security/bola/check_coverage.py
```

### Шаг 6: Проверьте audit log

Убедитесь, что в audit log есть запись о запрещённом доступе.

### Шаг 7: Запросите code review

Используйте чек-лист из раздела 13.

---

## 12. Примеры кода

### 12.1 TypeScript — API Gateway интеграционные тесты

```typescript
// tests/security/bola/api-gateway/bola-negative-tests.test.ts
// @hlv BOLA_CROSS_TENANT
// @hlv FORBIDDEN_CROSS_TENANT_ACCESS

import { describe, it, expect } from 'vitest';
import { createTestClient } from './test-client';
import { TestUsers, TestTenants, TestOntologies } from './test-fixtures';

const client = createTestClient();

describe('BOLA Cross-Tenant — Ontology endpoints', () => {
  const alice = TestUsers.alice_tenant_A;    // admin, tenant A
  const eve  = TestUsers.eve_tenant_B;        // editor, tenant B

  it('GET /api/v1/ontologies/{id} — tenant A user cannot fetch tenant B ontology', async () => {
    // @hlv BOLA_CROSS_TENANT
    const res = await client
      .asUser(alice)
      .get(`/api/v1/ontologies/${TestOntologies.tenant_B_ontology}`);

    expect(res.status).toBe(403);
    expect(res.body.error.code).toBe('FORBIDDEN_CROSS_TENANT_ACCESS');
  });

  it('POST /api/v1/ontologies — tenant A user cannot create ontology in tenant B', async () => {
    // @hlv BFLA
    const res = await client
      .asUser(alice)
      .post('/api/v1/ontologies')
      .send({ tenantId: TestTenants.tenant_B, name: 'malicious' });

    expect(res.status).toBe(403);
    expect(res.body.error.code).toBe('FORBIDDEN_CROSS_TENANT_ACCESS');
  });

  it('DELETE /api/v1/ontologies/{id} — viewer cannot delete ontology', async () => {
    // @hlv BFLA
    const bob = TestUsers.bob_tenant_A_viewer;
    const res = await client
      .asUser(bob)
      .delete(`/api/v1/ontologies/${TestOntologies.tenant_A_ontology}`);

    expect(res.status).toBe(403);
    expect(res.body.error.code).toBe('FORBIDDEN_INSUFFICIENT_ROLE');
  });
});

describe('BOLA IDOR — Guessable IDs', () => {
  it('sequential numeric ID returns 403, never 404', async () => {
    // @hlv IDOR_GUESSABLE
    const res = await client
      .asUser(TestUsers.alice_tenant_A)
      .get('/api/v1/ontologies/999999999');

    expect(res.status).toBe(403);
    expect(res.body.error.code).toBe('FORBIDDEN_OBJECT_NOT_FOUND_OR_ACCESS_DENIED');
  });

  it('invalid UUID returns 403, never 404', async () => {
    // @hlv IDOR_GUESSABLE
    const res = await client
      .asUser(TestUsers.alice_tenant_A)
      .get('/api/v1/ontologies/00000000-0000-0000-0000-000000000101');

    expect(res.status).toBe(403);
    expect(res.body.error.code).toBe('FORBIDDEN_OBJECT_NOT_FOUND_OR_ACCESS_DENIED');
  });
});
```

### 12.2 TypeScript — GraphQL negative тесты

```typescript
// tests/security/bola/api-gateway/graphql-bola.test.ts
// @hlv BOLA_CROSS_TENANT

import { describe, it, expect } from 'vitest';
import { createGraphQLClient } from './test-client';
import { TestUsers, TestOntologies } from './test-fixtures';

describe('GraphQL — Cross-tenant BOLA', () => {
  it('mutation updateOntology — tenant A cannot modify tenant B ontology', async () => {
    // @hlv BOLA_CROSS_TENANT
    const client = createGraphQLClient(TestUsers.alice_tenant_A);

    const mutation = `
      mutation UpdateOntology($id: ID!, $name: String!) {
        updateOntology(id: $id, input: { name: $name }) {
          id
        }
      }
    `;

    const res = await client.query(mutation, {
      id: TestOntologies.tenant_B_ontology,
      name: 'malicious rename',
    });

    expect(res.status).toBe(403);
    expect(res.body.errors[0].extensions.code).toBe('FORBIDDEN_CROSS_TENANT_ACCESS');
  });

  it('query ontology — viewer cannot query restricted ontology', async () => {
    // @hlv BOLA_CROSS_OBJECT
    const client = createGraphQLClient(TestUsers.bob_tenant_A_viewer);

    const query = `
      query GetOntology($id: ID!) {
        ontology(id: $id) { id name }
      }
    `;

    const res = await client.query(query, {
      id: TestOntologies.tenant_A_restricted_ontology,
    });

    expect(res.status).toBe(403);
    expect(res.body.errors[0].extensions.code).toBe('FORBIDDEN_INSUFFICIENT_ROLE');
  });
});
```

### 12.3 Python — Интеграционные тесты (для CI)

```python
# tests/security/bola/bola_negative_tests.py
# @hlv BOLA_CROSS_TENANT
# @hlv FORBIDDEN_CROSS_TENANT_ACCESS

import pytest
import requests
from fixtures import test_users, test_tenants, test_ontologies, auth_header

class TestBolaCrossTenant:
    """Cross-tenant BOLA: пользователь tenant_A запрашивает объекты tenant_B."""

    @pytest.mark.parametrize("endpoint,method,object_id_field", [
        ("/api/v1/ontologies/{ontologyId}", "GET", "ontologyId"),
        ("/api/v1/ontologies/{ontologyId}", "PUT", "ontologyId"),
        ("/api/v1/ontologies/{ontologyId}", "DELETE", "ontologyId"),
        ("/api/v1/branches/{branchId}", "GET", "branchId"),
        ("/api/v1/commits/{commitId}", "GET", "commitId"),
    ])
    def test_cross_tenant_object_access(self, base_url, endpoint, method, object_id_field,
                                         test_users, test_tenants, test_ontologies):
        # @hlv BOLA_CROSS_TENANT
        user = test_users["alice_tenant_A"]
        target_tenant = test_tenants["tenant_B"]

        path_params = {
            "tenantId": target_tenant["id"],
            "ontologyId": test_ontologies["tenant_B_ontology"]["id"],
            "branchId": f"branch_{target_tenant['id']}",
            "commitId": f"commit_{target_tenant['id']}",
        }

        url = base_url + endpoint.format(**path_params)
        headers = auth_header(user["token"])

        response = requests.request(method, url, headers=headers)

        assert response.status_code == 403, (
            f"Expected 403 for cross-tenant access, got {response.status_code}: {response.text}"
        )
        assert response.json()["error"]["code"] == "FORBIDDEN_CROSS_TENANT_ACCESS"

    def test_cross_tenant_graphql_mutation(self, base_url, test_users, test_ontologies):
        # @hlv BOLA_CROSS_TENANT
        user = test_users["alice_tenant_A"]
        target_ontology = test_ontologies["tenant_B_ontology"]

        mutation = """
            mutation UpdateOntology($id: ID!, $name: String!) {
                updateOntology(id: $id, input: {name: $name}) { id }
            }
        """

        response = requests.post(
            f"{base_url}/graphql",
            json={"query": mutation, "variables": {
                "id": target_ontology["id"],
                "name": "malicious"
            }},
            headers=auth_header(user["token"]),
        )

        data = response.json()
        assert response.status_code == 403 or data.get("errors", [{}])[0].get("extensions", {}).get("code") == "FORBIDDEN_CROSS_TENANT_ACCESS"


class TestBolaCrossObject:
    """Cross-object BOLA: пользователь без прав на объект внутри своего tenant."""

    @pytest.mark.parametrize("role,expected", [
        ("viewer", 403),
        ("commenter", 403),
    ])
    def test_insufficient_role_delete(self, base_url, test_users, test_ontologies, role, expected):
        # @hlv BOLA_CROSS_OBJECT
        user = test_users[f"user_{role}_tenant_A"]
        ontology = test_ontologies["tenant_A_ontology"]

        response = requests.delete(
            f"{base_url}/api/v1/ontologies/{ontology['id']}",
            headers=auth_header(user["token"]),
            params={"tenantId": user["tenant_id"]},
        )

        assert response.status_code == expected


class TestBflaPrivilegeEscalation:
    """BFLA: пользователь вызывает endpoint, требующий более высокую роль."""

    @pytest.mark.parametrize("endpoint,method,role", [
        ("/api/v1/users", "GET", "viewer"),           # только admin
        ("/api/v1/roles", "POST", "editor"),            # только admin
        ("/api/v1/backups", "POST", "editor"),          # только admin
        ("/api/v1/tenants/{tenantId}/settings", "PUT", "viewer"),  # только admin
    ])
    def test_admin_only_endpoints(self, base_url, endpoint, method, role,
                                   test_users, test_tenants):
        # @hlv BFLA
        user = test_users[f"user_{role}_tenant_A"]
        url = base_url + endpoint.format(tenantId=test_tenants["tenant_A"]["id"])

        body = {"name": "malicious"} if method in ("POST", "PUT") else None
        response = requests.request(
            method, url, json=body,
            headers=auth_header(user["token"]),
        )

        assert response.status_code == 403
        assert response.json()["error"]["code"] in (
            "FORBIDDEN_INSUFFICIENT_ROLE",
            "FORBIDDEN_ADMIN_ONLY",
        )


class TestIdorGuessable:
    """IDOR: подстановка предсказуемых ID."""

    @pytest.mark.parametrize("payload", [
        "999999999",
        "00000000-0000-0000-0000-000000000001",
        "ffffffff-ffff-ffff-ffff-ffffffffffff",
        "admin",
        "' OR 1=1 --",
        "../../etc/passwd",
    ])
    def test_guessable_ontology_id(self, base_url, test_users, payload):
        # @hlv IDOR_GUESSABLE
        user = test_users["alice_tenant_A"]

        response = requests.get(
            f"{base_url}/api/v1/ontologies/{payload}",
            headers=auth_header(user["token"]),
            params={"tenantId": user["tenant_id"]},
        )

        # Должен быть 403, не 404 (не раскрываем существование объекта)
        # и не 500 (не падаем на некорректном ID)
        assert response.status_code == 403, (
            f"Expected 403 for guessable ID {payload!r}, got {response.status_code}"
        )

    def test_sql_injection_in_ontology_id(self, base_url, test_users):
        # @hlv IDOR_GUESSABLE
        user = test_users["alice_tenant_A"]
        payloads = [
            "1; DROP TABLE ontologies; --",
            "' UNION SELECT * FROM users --",
            "1 AND 1=1",
        ]

        for payload in payloads:
            response = requests.get(
                f"{base_url}/api/v1/ontologies/{payload}",
                headers=auth_header(user["token"]),
                params={"tenantId": user["tenant_id"]},
            )
            # SQL injection не должен влиять на ответ
            assert response.status_code in (403, 400), (
                f"SQL injection payload {payload!r} caused {response.status_code}"
            )
```

### 12.4 Python — Fuzzing IDOR

```python
# tests/security/fuzzing/fuzz_idor.py
# @hlv IDOR_GUESSABLE

import requests
import uuid
import random
import string
from typing import Iterator

IDOR_PATTERNS = [
    # Числовые последовательности
    *[str(i) for i in range(1, 100)],
    *[str(i) for i in range(999990, 1000010)],
    # UUID варианты
    "00000000-0000-0000-0000-000000000000",
    str(uuid.uuid4()),  # случайный UUID
    "ffffffff-ffff-ffff-ffff-ffffffffffff",
    # Base64 варианты
    "AAAAAAAAAAA=",
    "AAAAAAAAAAA",
    # Специальные символы
    "null",
    "undefined",
    "true",
    "false",
    # Path traversal
    "..%2F..%2F..%2Fetc%2Fpasswd",
    # XSS
    "<script>alert(1)</script>",
]

def fuzz_endpoint(base_url: str, endpoint_template: str,
                  auth_token: str, tenant_id: str) -> Iterator[dict]:
    """
    Fuzzing IDOR: перебор предсказуемых ID для эндпоинта.

    Args:
        base_url: базовый URL API
        endpoint_template: шаблон с {id}, например /api/v1/ontologies/{id}
        auth_token: JWT токен
        tenant_id: tenant ID для query params

    Yields:
        dict с результатом каждого запроса
    """
    headers = {"Authorization": f"Bearer {auth_token}"}

    for pattern in IDOR_PATTERNS:
        url = base_url + endpoint_template.replace("{id}", pattern)
        params = {"tenantId": tenant_id} if tenant_id else {}

        try:
            response = requests.get(url, headers=headers, params=params, timeout=5)
        except requests.RequestException as e:
            yield {"pattern": pattern, "error": str(e)}
            continue

        yield {
            "pattern": pattern,
            "status": response.status_code,
            "body_length": len(response.text),
            "error_code": response.json().get("error", {}).get("code"),
        }

        # Если получили 200 — нашли уязвимость
        if response.status_code == 200:
            print(f"[!] VULNERABILITY: {pattern} returned 200 for {url}")


if __name__ == "__main__":
    import sys
    base_url = sys.argv[1]
    duration = int(sys.argv[2]) if len(sys.argv) > 2 else 300

    for result in fuzz_endpoint(
        base_url=base_url,
        endpoint_template="/api/v1/ontologies/{id}",
        auth_token="test_token",
        tenant_id="tenant_A",
    ):
        if result.get("status") == 200:
            print(f"VULNERABILITY: {result}")
```

---

## 13. Чек-лист для Code Review: "Что проверять в MR на предмет BOLA"

### 13.1 Проверка каждого нового эндпоинта

- [ ] **Есть ли авторизационная проверка?** — middleware/custom handler проверяет права на объект
- [ ] **Проверяется ли ownership на tenant level?** — `tenantId` из JWT == `tenantId` объекта
- [ ] **Проверяется ли ownership на object level?** — пользователь имеет роль/право на конкретный объект
- [ ] **Проверяется ли function-level authorization?** — роль пользователя позволяет вызывать данный метод
- [ ] **Используется ли универсальный 403?** — никогда `404`, `401` или `500` для чужих/несуществующих объектов
- [ ] **Проверены ли все параметры-идентификаторы?** — не только `id`, но и `tenantId`, `branchId`, `commitId` и т.д.
- [ ] **Нет ли hardcoded credentials или bypass?** — `if (user.role === 'admin')` вместо policy check

### 13.2 Проверка GraphQL резолверов

- [ ] **Каждый резолвер проверяет авторизацию** — нет "ленивых" резолверов без проверки
- [ ] **Batch-запросы не позволяют смешивать tenant** — один запрос не должен читать данные из разных tenant
- [ ] **N+1 не обходит авторизацию** — DataLoader не должен кешировать данные без проверки прав

### 13.3 Проверка тестов

- [ ] **Negative тест написан?** — YAML в тестовой документации
- [ ] **Покрыты все типы (A/B/C/D)?** — для каждого объектного ID
- [ ] **Проверен audit log?** — `audit_log_must_contain: true`
- [ ] **Нет 404/500 в expected?** — все negative тесты ожидают 403

### 13.4 Проверка infra

- [ ] **CI pipeline включает BOLA-тесты?** — job bola-negative-tests присутствует
- [ ] **SAST проверка добавлена?** — check-authz-coverage скрипт
- [ ] **Runtime monitoring настроен?** — метрики для 403 rate

---

## 14. YAML-файл для CI-валидации тестового покрытия

```yaml
# tests/security/bola/bola-coverage.yaml
# Служебный файл: описывает, какие тесты должны существовать

coverage:
  p0:
    - endpoint: /api/v1/tenants/{tenantId}
      methods: [GET, PUT, DELETE]
      required_types: [A, C]
    - endpoint: /api/v1/ontologies/{ontologyId}
      methods: [GET, POST, PUT, DELETE]
      required_types: [A, B, C, D]
    - endpoint: /api/v1/branches/{branchId}
      methods: [GET, POST, PUT, DELETE]
      required_types: [A, C]
    - endpoint: /api/v1/commits/{commitId}
      methods: [GET]
      required_types: [A, C]
    - endpoint: /api/v1/users/{userId}
      methods: [GET, PUT, DELETE]
      required_types: [C]
    - endpoint: /api/v1/roles
      methods: [POST, PUT, DELETE]
      required_types: [C]
    - endpoint: /api/v1/backups/{backupId}
      methods: [GET, POST, DELETE]
      required_types: [A, C]
    - endpoint: graphql (mutations with ontologyId)
      methods: [MUTATION]
      required_types: [A, B, D]
    - endpoint: graphql (queries with ontologyId)
      methods: [QUERY]
      required_types: [A, D]
    - endpoint: websocket (collaboration join)
      methods: [WS_JOIN]
      required_types: [A]
  p1:
    - endpoint: /api/v1/invites/{inviteId}
      methods: [GET]
      required_types: [A]
    - endpoint: /api/v1/sparql
      methods: [POST]
      required_types: [A]
    - endpoint: grpc internal
      methods: [RPC]
      required_types: [A]
```

---

## 15. Бенчмарк тестовой производительности

| Метрика | Target |
|---------|--------|
| Время прогона всех P0 тестов | < 2 min |
| Время прогона fuzzing | < 5 min |
| SAST scan | < 1 min |
| DAST scan (staging) | < 10 min |
| Общее время security stage в CI | < 15 min |

---

## История изменений

| Версия | Дата | Автор | Изменения |
|--------|------|-------|-----------|
| v1.0 | 2026-05-16 | Security Architect | Initial specification |
| v1.1 | 2026-07-25 | Agent | Добавлены BFLA-тесты для lowercase realm-ролей Keycloak:
  `TestKeycloak_LowercaseRealmRole_OwnerCanPost` (owner → POST /groups — 200),
  `TestKeycloak_LowercaseRealmRole_ViewerBlocked` (viewer → DELETE — 403).
  Исправлен case-sensitive lookup ролей в API Gateway (`auth.go`). |
