# Метрики качества совместной работы (Collaboration Quality Metrics)

| Атрибут | Значение |
|---------|----------|
| **ID** | REQ-FUN.INTEGRATION.collaboration-quality-metrics |
| **Уровень** | FUN |
| **Атрибут качества** | Functionality |
| **Приоритет** | P1 |
| **Статус** | УТВЕРЖДЕНО |
| **Источник** | — |
| **Критерии приёмки** | — |

---

## Контекст

VEDO Core использует Git-подобную модель совместной работы: параллельная разработка ведётся в разных ветках, конфликты разрешаются при слиянии (merge). Метрики качества оценивают эффективность этой модели.

Метрики **«95% правок без конфликтов»** и **«ложных конфликтов = 0»** должны быть измеримыми для release validation и customer-facing SLA.

Измерение проводится на Canonical Workload Profile из `human/artifacts/requirements/REQ-NFR.PERF.canonical-workload-profile.md` и имитирует реальную командную работу.

---

## Метрики

| Метрика | Определение | Целевое значение | Метод измерения |
|---------|-------------|------------------|-----------------|
| Правки без ручного разрешения конфликтов | Доля операций слияния (merge), которые завершились автоматически, без ручного вмешательства | ≥ 95% | Логирование операций слияния |
| Ложные конфликты | Система зафиксировала конфликт при слиянии веток, изменяющих семантически независимые части графа | 0 | Анализ дампов конфликтов + ручная проверка |

### Уточнение определений

**Правки без ручного разрешения конфликтов:**

В отличие от модели с блокировками (где конфликт возникает в момент редактирования), в Git-подобной модели конфликты обнаруживаются **только при слиянии веток**. Поэтому метрика измеряется как доля операций слияния, не потребовавших ручного вмешательства.

Формула:

```
conflict_free_percentage = (total_merges - manual_merges) × 100 / total_merges
```

Где:
- `total_merges` — общее количество операций слияния веток за период измерения.
- `manual_merges` — количество слияний, потребовавших ручного разрешения конфликтов через UI.

**Ложный конфликт:**

Фиксируется, если выполнены **все** условия:
- Система обнаружила конфликт при слиянии веток.
- Изменялись разные элементы онтологии (разные URI/ID).
- Между изменёнными элементами **нет семантической связи**: не parent/child, не domain/range, не inverse, не иные явные зависимости, зафиксированные в онтологии.
- В течение 5 секунд до/после конфликта не было изменений зависимых сущностей (проверяется по логам).

**Не считаются ложными конфликтами** конфликты при изменении:
- Одного класса с его родительским классом.
- Свойства и его domain или range.
- Класса и индивида, принадлежащего этому классу.
- Связанных inverse-свойств.

---

## Нагрузка для измерения

Тестирование проводится на каноническом профиле: 1M axioms / 100k individuals / 5 параллельных редакторов (работающих в разных ветках).

Файл сценария: `collaboration-workload.yaml`.

```yaml
workload:
  ontology_size: "1M axioms / 100k individuals"
  duration: "8 часов (рабочий день)"
  users: 5
  operations_per_user_per_hour: 60

  operations:
    - type: "create_class"
      probability: 0.2
    - type: "edit_class"
      probability: 0.3
    - type: "delete_class"
      probability: 0.05
    - type: "create_individual"
      probability: 0.2
    - type: "edit_property"
      probability: 0.15
    - type: "create_branch"
      probability: 0.05
    - type: "merge_branch"
      probability: 0.05
      trigger: "after_10_commits"

  conflict_scenarios:
    - name: "true_positive_edit_same_class"
      probability: 0.1
      description: "Два пользователя в разных ветках правят один класс"
    - name: "true_positive_parent_change"
      probability: 0.05
      description: "Один меняет родителя класса, другой правит свойства того же класса"
    - name: "false_conflict_candidate"
      probability: 0.2
      description: "Правят разные классы в разных ветках -> НЕ ДОЛЖНО БЫТЬ КОНФЛИКТА ПРИ СЛИЯНИИ"
```

---

## Инструменты измерения

Основной инструмент: k6 для workload execution и Loki для логирования результатов.

Каждая операция слияния и каждый конфликт записываются в structured logs.

**Минимальные поля события (для слияния):**

| Поле | Тип | Описание |
|------|-----|----------|
| `merge_id` | string | Уникальный идентификатор операции слияния |
| `source_branch` | string | Исходная ветка |
| `target_branch` | string | Целевая ветка |
| `user` | string | Инициатор слияния |
| `conflict` | boolean | Был ли конфликт |
| `manual_resolution` | boolean | Потребовалось ли ручное разрешение |
| `false_conflict_candidate` | boolean | Подозрение на ложный конфликт |
| `duration_ms` | integer | Длительность операции |
| `timestamp` | timestamp | Время выполнения |

**Пример k6-фрагмента:**

```javascript
const mergeResponse = http.post(
  `${API_URL}/branches/${sourceBranch}/merge`,
  JSON.stringify({ target: targetBranch }),
  { headers: { Authorization: `Bearer ${token}` } }
);

const conflict = mergeResponse.status === 409;
const manualResolution = mergeResponse.status === 409 && 
                        mergeResponse.json().requires_manual === true;

const falseConflictCandidate = isFalseConflictCandidate(
  sourceBranch, 
  targetBranch, 
  mergeResponse
);

logToLoki({
  merge_id: uuid(),
  source_branch: sourceBranch,
  target_branch: targetBranch,
  user: user.email,
  conflict,
  manual_resolution: manualResolution,
  false_conflict_candidate: falseConflictCandidate,
  duration_ms: mergeResponse.timings.duration,
  timestamp: new Date().toISOString(),
});
```

---

## Период измерения

| Сценарий | Период | Минимальный объём выборки |
|----------|--------|---------------------------|
| Release validation (CI) | Синтетический workload | ≥ 100 операций слияния |
| Production-статистика для SLA | 30 календарных дней | ≥ 500 операций слияния |
| Customer-facing отчёт (Enterprise) | 30 календарных дней | Все операции слияния за период |

Для release CI допускается сокращённый synthetic workload, но release report должен явно указывать объём выборки.

---

## Расчёт метрик

### Основной запрос

```sql
WITH stats AS (
  SELECT
    COUNT(*) as total_merges,
    SUM(CASE WHEN manual_resolution = true THEN 1 ELSE 0 END) as manual_merges,
    SUM(CASE WHEN conflict = true AND false_conflict_candidate = true THEN 1 ELSE 0 END) as false_conflicts
  FROM merge_logs
  WHERE timestamp BETWEEN '2025-01-01' AND '2025-01-31'
)
SELECT
  total_merges,
  manual_merges,
  false_conflicts,
  ROUND((total_merges - manual_merges) * 100.0 / total_merges, 2) as auto_merge_percentage,
  false_conflicts
FROM stats;
```

### Детализация ложных конфликтов

Если `false_conflicts > 0`, отчёт должен включать детализацию:

```sql
SELECT 
  merge_id,
  source_branch,
  target_branch,
  timestamp,
  'REQUIRES_MANUAL_REVIEW' as action
FROM merge_logs
WHERE conflict = true AND false_conflict_candidate = true
ORDER BY timestamp DESC;
```

---

## CI-гейт (GitLab CI)

В GitLab CI добавляется job `false-conflict-test`, который прогоняет workload и проверяет метрики.

```yaml
false-conflict-test:
  stage: test
  script:
    - k6 run k6-collaboration-test.js
    - python scripts/analyze-merge-logs.py --start "$START_TIME" --end "$END_TIME"
  artifacts:
    reports:
      junit: collaboration-report.xml
    paths:
      - collaboration-metrics.json
```

**Release gate считается failed, если:**

| Условие | Действие |
|---------|----------|
| `auto_merge_percentage < 95` | Блокировка релиза |
| `false_conflicts > 0` | Блокировка релиза, требуется ручной анализ |

**Исключения (требуют формального approval):**

- `false_conflicts > 0`, но доказано, что это истинный конфликт (ошибка классификации в тесте).
- Недостаточный объём выборки (< 100 слияний) — тогда метрика помечается как `insufficient_data`.

---

## Отображение метрик для заказчиков

### Enterprise SLA (customer-visible)

Для Enterprise-клиентов предоставляется ежемесячный отчёт, включающий:

| Метрика | Значение за период | Целевое значение |
|---------|---------------------|------------------|
| Доля автоматических слияний | 96.5% | ≥ 95% |
| Ложные конфликты | 0 | 0 |
| Количество слияний за период | 1247 | — |
| Количество ручных разрешений | 44 | — |

### Формат отчёта (выписка)

```yaml
report_period: "2026-01-01 to 2026-01-31"
total_merges: 1247
manual_merges: 44
auto_merge_percentage: 96.5
false_conflicts: 0
target_auto_merge_percentage: 95
target_false_conflicts: 0
status: "PASSED"
```

---

## Объяснение для заказчиков

VEDO Core использует Git-подобную модель совместной работы. Это означает:

- Каждый инженер работает в своей ветке.
- Изменения объединяются через Merge Request.
- Конфликты возникают только при слиянии веток, а не в момент редактирования.
- Конфликт требует ручного разрешения через семантический diff.

**Метрики качества:**

1. **Доля автоматических слияний (≥ 95%)** — показывает, как часто система может объединить изменения без участия человека.
2. **Ложные конфликты (0)** — гарантирует, что система не маркирует как конфликт изменения, которые семантически независимы.

Каждый релиз прогоняется через нагрузочный тест с 5 параллельными редакторами на онтологии из 1 миллиона аксиом. Отчёт доступен в CI artifacts. Для Enterprise-клиентов предоставляется ежемесячный отчёт по production-метрикам.

---

## Статус

**Решено.** Метрики утверждены для release validation и Enterprise SLA.

---
