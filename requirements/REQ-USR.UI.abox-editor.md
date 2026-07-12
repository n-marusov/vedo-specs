# Редактор ABox — управление индивидами (экземплярами классов)

| Атрибут | Значение |
|---------|----------|
| **ID** | REQ-USR.UI.abox-editor |
| **Уровень** | USR |
| **Атрибут качества** | Functionality |
| **Приоритет** | P0 |
| **Статус** | УТВЕРЖДЕНО |
| **Источник** | — |
| **Критерии приёмки** | — |

---


## Обзор

ABox Editor обеспечивает работу с индивидами (экземплярами классов) в онтологии. Индивиды представляют конкретные сущности реального мира, описанные в терминах TBox (классов и свойств). Этот модуль дополняет TBox Editor, позволяя наполнять онтологию фактами.

## Ключевые Возможности

### 1. Управление индивидами (CRUD)

**Форма создания индивида:**

```typescript
interface IndividualCreation {
  id: string;           // URI-совместимый ID
  label: string;         // rdfs:label — обязательно
  comment?: string;      // rdfs:comment
  types: string[];       // owl:type (классы, которым принадлежит индивид)
  properties?: {
    objectProperty?: { [propertyId]: string[] };  // ссылки на других индивидов
    datatypeProperty?: { [propertyId]: value };    // литеральные значения
  };
  deprecated?: boolean;   // vedo:deprecated
}
```

**Операции:**
- Внешние REST URL ниже публикуются только API Gateway; Ontology Service выполняет эти операции через внутренний gRPC/protobuf contract и не должен иметь функциональные REST-хендлеры для ABox CRUD.
- Create → POST /api/v1/ontologies/{id}/individuals
- Read → GET /api/v1/ontologies/{id}/individuals/{individualId}
- Update → PATCH /api/v1/ontologies/{id}/individuals/{individualId}
- Delete → DELETE /api/v1/ontologies/{id}/individuals/{individualId}

### 2. Назначение типа (owl:type)

**Семантика:**
- Индивид может принадлежать нескольким классам (`owl:type` = `rdf:type`)
- Поддержка множественного наследования типов
- Автоматический вывод типов через Reasoner (опционально)

**Возможности UI:**
- Выпадающий список классов из TBox
- Поиск по классам
- Отображение унаследованных свойств от всех типов

### 3. Назначение значений свойств

**Object Properties (ссылки на индивидов):**

```typescript
interface ObjectPropertyValue {
  propertyId: string;    // ID свойства из TBox
  value: string;          // ID целевого индивида
  qualifiers?: {          // Опциональные квалификаторы (qualified cardinality)
    propertyId: string;
    value: string;
  }[];
}
```

**Datatype Properties (литеральные значения):**

```typescript
interface DatatypePropertyValue {
  propertyId: string;    // ID свойства из TBox
  value: string | boolean | number | Date;
  language?: string;     // Для xsd:string с многоязычностью
}
```

**Примеры:**

```json
POST /api/v1/ontologies/{ontologyId}/individuals
{
  "id": "ТЭС-Новосибирская-2024",
  "label": {"ru": "Новосибирская ТЭЦ-2"},
  "types": ["ТепловаяЭлектростанция", "ЭнергетическийОбъект"],
  "properties": {
    "objectProperty": {
      "имеетРегион": ["Регион-Сибирь"],
      "принадлежитКомпании": ["ЭнергетическаяКомпания-СИБУР"]
    },
    "datatypeProperty": {
      "годВводаВЭксплуатацию": 1980,
      "мощностьМВт": 100
    }
  }
}
```

### 4. Поиск и фильтрация индивидов

**Возможности поиска:**
- Полнотекстовый поиск по label, comment
- Фильтрация по типам (классам)
- Фильтрация по значениям свойств
- Пагинация для больших наборов индивидов

**Интеграция с представлением графа:**
- Индивиды отображаются как узлы того же типа, что и классы (с визуальным отличием)
- Связи между индивидами через ObjectProperty отображаются как рёбра графа

### 5. Массовые операции

**Импорт из CSV/JSON:**

```typescript
interface BulkImport {
  format: 'csv' | 'json';
  mapping: {
    sourceColumn: string;      // Колонка из файла
    targetProperty: string;     // Свойство в онтологии
    transform?: 'uppercase' | 'lowercase' | 'trim' | 'date';
  }[];
  defaultType?: string;          // Класс по умолчанию для всех импортируемых
}
```

**Экспорт в CSV/JSON:**
- Экспорт выбранных индивидов
- Настраиваемые колонки (какие свойства включить)

### 6. Обработка ошибок

**Дублирующийся ID:**
```
Error: "Individual with such ID already exists"
Resolution: Изменить ID или использовать namespace для различия
```

**Некорректное назначение типа:**
```
SPARQL check: Проверка domain свойства
Error: "Property {prop} cannot be applied to class {class}"
```

**Ссылочная целостность:**
```
API: DELETE with VerifyReference=1
Error: "Individual {ind} is referenced by {count} other individuals"
```

### 7. Мягкое удаление и архивирование

**Мягкое удаление:**
- `vedo:archive = true` — индивид скрывается из UI
- Сохраняется в графе для ссылочной целостности
- Можно восстановить через админку

**Жёсткое удаление:**
- Удаление индивида и всех его ссылок
- Требует подтверждения при наличии зависимостей

## События

| Событие | Триггер | Действие |
|-------|---------|--------|
| `IndividualCreated` | Save в форме индивида | Создать узел в Neo4j, сохранить коммит |
| `IndividualUpdated` | Изменение полей | Обновить в Neo4j, сохранить коммит |
| `IndividualDeleted` | Подтверждение удаления | Мягкое или жёсткое удаление |
| `TypeAdded` | Добавление типа | Обновить owl:type, сохранить коммит |
| `TypeRemoved` | Удаление типа | Обновить owl:type, сохранить коммит |
| `PropertyValueSet` | Заполнение свойства | Добавить/обновить значение в графе |
| `PropertyValueRemoved` | Очистка свойства | Удалить значение из графа |

## Примеры API

**Получить все индивиды класса:**

```json
GET /api/v1/ontologies/{ontologyId}/individuals?type=ТепловаяЭлектростанция
{
  "items": [
    {"id": "ТЭС-Новосибирская-2024", "label": "Новосибирская ТЭЦ-2"},
    {"id": "ТЭС-Саратовская-2024", "label": "Саратовская ТЭЦ-5"}
  ],
  "total": 2,
  "page": 1,
  "pageSize": 50
}
```

**Обновить значение свойства:**

```json
PATCH /api/v1/ontologies/{ontologyId}/individuals/{individualId}/properties
{
  "propertyId": "мощностьМВт",
  "value": 120,
  "operation": "set"
}
```

## Зависимости

- **Neo4j**: Хранение ABox (индивиды, их типы, связи между индивидами)
- **PostgreSQL**: Метаданные (createdBy, createdAt, version)
- **Redis**: Блокировки при редактировании индивида
- **Versioning Service**: Сохранение коммитов при изменениях
- **TBox Editor**: Референс на классы и свойства (TBox должен существовать)

## Проектные решения

| Вопрос | Решение |
|--------|---------|
| OWL Reasoner для вывода типов | Только через API, будущая возможность |
| `owl:sameAs` для слияния индивидов | Будущая возможность |
| Квота на количество индивидов | 100,000 на онтологию (по умолчанию) |
