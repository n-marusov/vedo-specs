# ADR-DES.DATA.owl-validation-strategy — Стратегия валидации OWL-онтологий

**Статус:** ПРЕДЛОЖЕНО
**Дата:** 2026-07-15

## Контекст

VEDO Hub генерирует OWL-онтологии через LLM (NL→OWL, F14.1) и принимает их через импорт (F8). LLM могут генерировать синтаксически невалидный OWL, циклические иерархии, дублирующиеся IRI и другие дефекты (LLM hallucinations in code generation, исследования 2024-2025).

**Проблемы:**

1. **Синтаксические ошибки:** LLM генерирует невалидный OWL (незакрытые теги, невалидные IRI, ошибочные конструкции)
2. **Циклические иерархии:** Классы могут ссылаться друг на друга через `rdfs:subClassOf`, создавая логическую несостоятельность
3. **Дублирующиеся IRI:** LLM может создать два класса с одинаковым IRI
4. **Ссылочная целостность:** Ссылки на несуществующие сущности
5. **Некорректные типы данных:** DatatypeProperty могут использовать неподдерживаемые XSD-типы
6. **Неподдерживаемые OWL-конструкции:** OWL Full конструкции, не поддерживаемые VEDO Core

**Требуется** стратегия валидации, которая:
1. Выявляет все ошибки до показа пользователю
2. Разделяет ошибки на блокирующие и информационные
3. Предоставляет понятную обратную связь
4. Позволяет пользователю исправлять ошибки интерактивно
5. Применяется единообразно для генерации, импорта и Pull Request

## Требование-источник

- `REQ-FUN.API.owl-syntax-validation`
- `REQ-FUN.API.owl-no-cycles`
- `REQ-FUN.API.owl-unique-iri`
- `REQ-FUN.API.owl-domain-range`
- `REQ-FUN.API.owl-supported-constructs`
- `REQ-FUN.API.owl-labels`
- `REQ-FUN.API.owl-reference-integrity`
- `REQ-FUN.API.owl-datatype-validation`
- `REQ-USR.UI.validation-feedback`
- `REQ-USR.UI.validation-fix`
- `REQ-NFR.API.validation-latency`
- `REQ-FUN.API.validation-import`
- `REQ-FUN.API.validation-pr`

## Решение

Внедрить **OWL Validation Pipeline** — компонент для валидации OWL-онтологий.

### 1. Архитектурная схема

```
┌─────────────────────────────────────────────────────────────────────────┐
│                          UI Layer                                      │
│  ┌───────────────────────────────────────────────────────────────────┐ │
│  │                    Validation Report UI                           │ │
│  │  ┌─────────────────────────────────────────────────────────────┐ │ │
│  │  │  🔴 Блокирующие ошибки (N)                                  │ │ │
│  │  │  ┌─────────────────────────────────────────────────────┐    │ │ │
│  │  │  │  ❌ Синтаксическая ошибка в строке 42               │    │ │ │
│  │  │  │  ❌ Циклическая иерархия: Car → Vehicle → Car      │    │ │ │
│  │  │  └─────────────────────────────────────────────────────┘    │ │ │
│  │  ├─────────────────────────────────────────────────────────────┤ │ │
│  │  │  🟡 Информационные предупреждения (M)                       │ │ │
│  │  │  ┌─────────────────────────────────────────────────────┐    │ │ │
│  │  │  │  ⚠️ Свойство "hasEngine" ссылается на Engine       │    │ │ │
│  │  │  │  ⚠️ Класс "Vehicle" не имеет rdfs:label           │    │ │ │
│  │  │  └─────────────────────────────────────────────────────┘    │ │ │
│  │  └─────────────────────────────────────────────────────────────┘ │ │
│  │  [Исправить ошибки]  [Продолжить (с предупреждениями)]          │ │
│  └───────────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                          API Gateway                                   │
│  ┌───────────────────────────────────────────────────────────────────┐ │
│  │                    OWL Validation Pipeline                        │ │
│  │                                                                   │ │
│  │  ┌─────────────────────────────────────────────────────────────┐ │ │
│  │  │              Stage 1: Parser & Pre-processing               │ │ │
│  │  │  1. Parse OWL (Turtle/RDF/XML/JSON-LD)                     │ │ │
│  │  │  2. Build internal model (classes, properties, axioms)      │ │ │
│  │  │  3. Extract all IRI references                              │ │ │
│  │  └─────────────────────────────────────────────────────────────┘ │ │
│  │         │                                                        │ │
│  │         ▼                                                        │ │
│  │  ┌─────────────────────────────────────────────────────────────┐ │ │
│  │  │              Stage 2: Blocking Validations                  │ │ │
│  │  │  1. Syntax validation (OWL API)                            │ │ │
│  │  │  2. Cycle detection (DFS on rdfs:subClassOf)               │ │ │
│  │  │  3. IRI uniqueness                                         │ │ │
│  │  └─────────────────────────────────────────────────────────────┘ │ │
│  │         │                                                        │ │
│  │         ▼                                                        │ │
│  │  ┌─────────────────────────────────────────────────────────────┐ │ │
│  │  │              Stage 3: Informational Validations             │ │ │
│  │  │  1. Domain/range consistency                               │ │ │
│  │  │  2. Supported constructs check                             │ │ │
│  │  │  3. Label presence                                         │ │ │
│  │  │  4. Reference integrity                                    │ │ │
│  │  │  5. Datatype validation                                    │ │ │
│  │  └─────────────────────────────────────────────────────────────┘ │ │
│  │         │                                                        │ │
│  │         ▼                                                        │ │
│  │  ┌─────────────────────────────────────────────────────────────┐ │ │
│  │  │              Stage 4: Report Generation                     │ │ │
│  │  │  1. Aggregated results                                      │ │ │
│  │  │  2. Error metadata (location, suggestion, fix action)      │ │ │
│  │  │  3. Export to structured format (JSON)                     │ │ │
│  │  └─────────────────────────────────────────────────────────────┘ │ │
│  └───────────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                     Ontology Service                                   │
│  (Target ontology for validation)                                     │
└─────────────────────────────────────────────────────────────────────────┘
```

### 2. Классификация валидаций

| Категория | Валидации | Действие | Критичность |
|-----------|-----------|----------|-------------|
| **🔴 Блокирующие** | Синтаксис, циклы, дубликаты IRI | Запрет показа/сохранения | P0 |
| **🟡 Информационные** | Domain/range, OWL-конструкции, label, ссылки, типы данных | Показ с предупреждениями | P1 |

**Блокирующие валидации:**

| Валидация | Алгоритм | Ошибка при |
|-----------|----------|------------|
| **Синтаксис** | OWL API / RDF4J парсер | Любая синтаксическая ошибка |
| **Циклы** | DFS на `rdfs:subClassOf` графе | Обнаружение цикла |
| **Уникальность IRI** | Hash-таблица всех IRI | Две сущности с одинаковым IRI |

**Информационные валидации:**

| Валидация | Алгоритм | Предупреждение при |
|-----------|----------|-------------------|
| **Domain/range** | Проверка существования классов | Domain/range указывает на несуществующий класс |
| **Поддерживаемые конструкции** | Белый список OWL-конструкций | Использование неподдерживаемой конструкции |
| **Label** | Проверка наличия `rdfs:label` | Сущность без label |
| **Ссылочная целостность** | Проверка всех IRI-ссылок | Ссылка на несуществующую сущность |
| **Типы данных** | Проверка XSD-типов | Некорректный тип данных |

### 3. Детальный алгоритм валидации

```python
class OWLValidator:
    def validate(self, owl_content: str, format: str) -> ValidationResult:
        """Выполняет полную валидацию OWL-онтологии."""

        # Stage 1: Parser & Pre-processing
        try:
            model = self.parser.parse(owl_content, format)
        except ParseException as e:
            return ValidationResult(
                blocking_errors=[ValidationError(
                    type="syntax_error", message=str(e),
                    location=self.get_location(e),
                    suggestion="Исправьте синтаксическую ошибку в указанном месте"
                )], warnings=[], status="FAILED")

        # Stage 2: Blocking Validations
        blocking_errors = []

        # 2.1 Cycle detection (DFS on rdfs:subClassOf)
        cycles = self.detect_cycles(model)
        for cycle in cycles:
            blocking_errors.append(ValidationError(
                type="cyclic_hierarchy",
                message=f"Цикл: {' → '.join(cycle)}",
                location=f"Classes: {', '.join(cycle)}",
                suggestion=f"Удалите или измените одну из связей",
                fix_action={"type": "remove_cycle", "cycle": cycle}))

        # 2.2 IRI uniqueness
        duplicate_iris = self.find_duplicate_iris(model)
        for iri, entities in duplicate_iris.items():
            blocking_errors.append(ValidationError(
                type="duplicate_iri",
                message=f"Дубликат IRI: {iri}",
                location=f"Entities: {', '.join(entities)}",
                suggestion="Переименуйте дублирующиеся сущности",
                fix_action={"type": "rename_iri", "iri": iri, "entities": entities}))

        if blocking_errors:
            return ValidationResult(blocking_errors=blocking_errors,
                warnings=[], status="BLOCKED")

        # Stage 3: Informational Validations
        warnings = []

        # 3.1 Domain/range consistency
        for prop in model.object_properties:
            if prop.domain and prop.domain not in model.classes:
                warnings.append(ValidationWarning(
                    type="domain_not_found",
                    message=f"Свойство '{prop.iri}': domain '{prop.domain}' не существует",
                    suggestion="Создайте класс или измените domain",
                    fix_action={"type": "create_class", "class_iri": prop.domain}))
            if prop.range and prop.range not in model.classes:
                warnings.append(ValidationWarning(
                    type="range_not_found",
                    message=f"Свойство '{prop.iri}': range '{prop.range}' не существует",
                    suggestion="Создайте класс или измените range",
                    fix_action={"type": "create_class", "class_iri": prop.range}))

        # 3.2 Supported constructs
        for axiom in model.axioms:
            if axiom.type not in SUPPORTED_CONSTRUCTS:
                warnings.append(ValidationWarning(
                    type="unsupported_construct",
                    message=f"Неподдерживаемая конструкция: {axiom.type}",
                    suggestion="Удалите или замените на поддерживаемую",
                    fix_action={"type": "remove_axiom", "axiom_id": axiom.id}))

        # 3.3 Label presence
        for entity in model.all_entities:
            if not entity.label:
                warnings.append(ValidationWarning(
                    type="missing_label",
                    message=f"Сущность '{entity.iri}' без rdfs:label",
                    suggestion="Добавьте rdfs:label для улучшения навигации",
                    fix_action={"type": "add_label", "entity_iri": entity.iri}))

        # 3.4 Reference integrity
        for ref in model.references:
            if ref.target not in model.all_iris:
                warnings.append(ValidationWarning(
                    type="missing_reference",
                    message=f"Ссылка на несуществующую сущность: '{ref.target}'",
                    suggestion="Создайте сущность или исправьте ссылку",
                    fix_action={"type": "create_entity",
                        "entity_iri": ref.target, "entity_type": "Class"}))

        # 3.5 Datatype validation
        for prop in model.datatype_properties:
            if prop.range not in SUPPORTED_XSD_TYPES:
                warnings.append(ValidationWarning(
                    type="invalid_datatype",
                    message=f"'{prop.iri}': некорректный тип '{prop.range}'",
                    suggestion=f"Замените на: {', '.join(SUPPORTED_XSD_TYPES[:3])}",
                    fix_action={"type": "change_datatype",
                        "property_iri": prop.iri, "suggested_types": SUPPORTED_XSD_TYPES}))

        return ValidationResult(blocking_errors=[],
            warnings=warnings, status="OK" if not warnings else "WARNINGS")
```

### 4. Интерактивное исправление

**Fix actions (действия по исправлению):**

| Тип ошибки | Fix action | Параметры |
|------------|-----------|-----------|
| Циклическая иерархия | `remove_cycle` | `cycle: List[str]` — список классов в цикле |
| Дубликат IRI | `rename_iri` | `iri: str`, `entities: List[str]` |
| Отсутствующий domain | `create_class` | `class_iri: str` |
| Неподдерживаемая конструкция | `remove_axiom` | `axiom_id: str` |
| Отсутствующий label | `add_label` | `entity_iri: str` |
| Отсутствующая ссылка | `create_entity` | `entity_iri: str`, `entity_type: str` |
| Некорректный тип данных | `change_datatype` | `property_iri: str`, `suggested_types: List[str]` |

### 5. Производительность

| Размер онтологии | p50 | p95 | Режим |
|------------------|-----|-----|-------|
| ≤ 10 классов | ≤ 500 мс | ≤ 1 с | Синхронный |
| 11-50 классов | ≤ 2 с | ≤ 4 с | Синхронный |
| 51-100 классов | ≤ 5 с | ≤ 10 с | Синхронный (с прогресс-баром) |
| > 100 классов | ≤ 10 с | ≤ 15 с | Асинхронный (фоновая) |

**Оптимизации:**
1. **Досрочное завершение:** Прерывание при обнаружении первой блокирующей ошибки
2. **Кэширование:** Срок хранения 5 минут для неизменённых онтологий
3. **Потоковый парсер:** Для больших файлов избегает полной загрузки в память

### 6. Применение валидации

| Контекст | Блокирующие | Информационные | Поведение |
|----------|------------|----------------|-----------|
| **NL→OWL генерация** | ✅ | ✅ | 🔴 → без предпросмотра, 🟡 → предпросмотр с предупреждениями |
| **Импорт OWL (F8)** | ✅ | ✅ | 🔴 → импорт остановлен, 🟡 → отчёт в результатах |
| **Pull Request (F13.2)** | ✅ | ✅ | 🔴 → слияние заблокировано, 🟡 → комментарий в PR |
| **Ручное редактирование** | ✅ | ❌ | 🔴 → запрет сохранения |

## Рассмотренные альтернативы

| Альтернатива | Причина отклонения |
|--------------|--------------------|
| Только синтаксическая валидация | Циклические иерархии и дубликаты IRI остаются незамеченными |
| Все валидации блокирующие | Негативный UX — пользователь не может продолжить из-за незначительных проблем |
| Все валидации информационные | Риск сохранения невалидных онтологий |
| Валидация только при сохранении | Раннее обнаружение (до показа пользователю) даёт лучший UX |
| Интерактивное исправление через LLM | Непредсказуемость — детерминированные исправления надёжнее |

## Последствия

**Положительные последствия:**

1. **Качество данных:** Блокирующие валидации предотвращают сохранение невалидных онтологий.
2. **Прозрачность:** Пользователь видит все проблемы до сохранения — нет скрытых ошибок.
3. **Контроль:** Пользователь может исправлять ошибки интерактивно, не возвращаясь к NL-описанию.
4. **Единообразие:** Одинаковые валидации для генерации, импорта и Pull Request.
5. **Гибкость:** Разделение на блокирующие и информационные, конфигурируемый белый список конструкций.

**Отрицательные последствия:**

1. **Сложность:** Дополнительный компонент Validation Pipeline требует разработки и поддержки.
2. **Задержка:** Валидация добавляет задержку (≤ 500 мс для типовой онтологии из 10 классов).
3. **Ложные срабатывания:** Валидный OWL может быть помечен как ошибочный (редко, для граничных случаев).
4. **Поддержка:** Белый список OWL-конструкций требует актуализации при расширении функциональности.

**Меры снижения рисков:**

1. **Сложность:** Чёткая модульная архитектура с разделением на этапы (Stage 1-4).
2. **Задержка:** Досрочное завершение при критической ошибке, кэширование, асинхронность для больших онтологий.
3. **Ложные срабатывания:** Возможность отключить отдельные валидации через конфигурацию.
4. **Поддержка:** Конфигурируемый белый список OWL-конструкций в YAML.

**Риски:**

1. **Новый тип ошибки:** LLM может генерировать ошибки, не покрытые существующими валидациями.
   - **Смягчение:** Расширяемая архитектура — добавление новой валидации через регистрацию в pipeline.
2. **Производительность:** Валидация больших онтологий может быть медленной.
   - **Смягчение:** Асинхронная валидация для онтологий > 100 классов.
3. **Пользовательское сопротивление:** Блокирующие валидации могут раздражать.
   - **Смягчение:** Чёткие сообщения с конкретными предложениями по исправлению, интерактивные fix actions.

## Связанные ADR

- `ADR-IMPL.SECURITY.parser-query-fuzz-gates-mandate` — Фаззинг-гейты парсеров (используется для синтаксической валидации)
- `ADR-DES.UI.error-feedback-strategy` — Обратная связь об ошибках (шаблон для Validation Report UI)
- `ADR-DES.UI.data-loss-prevention-strategy` — Защита от потери данных (блокирующие валидации как часть DLP)
- `ADR-DES.API.llm-policy-router-strategy` — стратегия роутинга LLM-запросов (контекст генерации)

## Чек-лист реализации

- [ ] Реализация OWL Validation Pipeline (Stage 1-4)
- [ ] Реализация блокирующих валидаций (синтаксис, циклы, уникальность IRI)
- [ ] Реализация информационных валидаций (domain/range, конструкции, label, ссылки, типы)
- [ ] UI Validation Report с разделением на ошибки/предупреждения
- [ ] Интерактивное исправление ошибок (7 типов fix actions)
- [ ] Интеграция с NL→OWL генерацией (F14.1)
- [ ] Интеграция с импортом OWL (F8)
- [ ] Интеграция с Pull Request (F13.2 — семантический линтер)
- [ ] Конфигурация валидаций (YAML, hot reload)
- [ ] Документация User Guide: работа с отчётом валидации
- [ ] Тестирование: boundary cases (пустая онтология, 1000 классов, все типы ошибок)

---
