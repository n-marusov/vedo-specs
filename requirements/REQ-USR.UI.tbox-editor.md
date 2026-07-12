# Редактор TBox — редактор онтологий (Classes, Properties, Annotations)

| Атрибут | Значение |
|---------|----------|
| **ID** | REQ-USR.UI.tbox-editor |
| **Уровень** | USR |
| **Атрибут качества** | Functionality |
| **Приоритет** | P0 |
| **Статус** | УТВЕРЖДЕНО |
| **Источник** | — |
| **Критерии приёмки** | — |

---


## Обзор

Редактор TBox позволяет управлять терминологией онтологии: классами, свойствами (ObjectProperty, DatatypeProperty) и аннотациями. Базируется на опыте АрхиГраф.Мир.

## Ключевые возможности

### 1. Управление Class (CRUD)

**Форма создания класса:**

```typescript
interface ClassCreation {
  id: string;           // URI-совместимый ID (автогенерация или ручной ввод)
  label: string;        // rdfs:label — обязательно
  comment?: string;     // rdfs:comment
  parents?: string[];    // rdfs:subClassOf — множественное наследование
  isAbstract?: boolean;  // owl:Class (abstract flag via annotation)
  deprecated?: boolean;  // owl:deprecated
}
```

**Алгоритм генерации ID (как в АрхиГраф):**
1. Наименование разбивается на слова
2. Начальная буква каждого слова → upper
3. Конкатенация без пробелов
4. Не-ASCII символы удаляются

Примеры:
- "Тепловая электростанция" → `ТепловаяЭлектростанция`
- "Насос К70-80-100" → `НасосК70-80-100`

**Операции:**
- Внешние REST URL ниже публикуются только API Gateway; Ontology Service выполняет эти операции через внутренний gRPC/protobuf contract и не должен иметь функциональные REST-хендлеры для TBox CRUD.
- Создание → POST /api/v1/ontologies/{id}/classes
- Чтение → GET /api/v1/ontologies/{id}/classes/{classId}
- Обновление → PATCH /api/v1/ontologies/{id}/classes/{classId}
- Удаление → DELETE /api/v1/ontologies/{id}/classes/{classId}

### 2. Управление ObjectProperty

```typescript
interface ObjectPropertyCreation {
  id: string;
  label: string;
  comment?: string;
  domain: string[];      //owl:Class(es) where property applies
  range: string[];       //owl:Class(es) that can be values
  characteristics?: {
    transitive?: boolean;
    symmetric?: boolean;
    functional?: boolean;
  };
  inverseOf?: string;    // инверсное свойство
}
```

**Ограничения:**
- domain и range — массивы классов
- Поддержка кардинальности (min/max cardinality) через аннотации

### 3. Управление DatatypeProperty

```typescript
interface DatatypePropertyCreation {
  id: string;
  label: string;
  comment?: string;
  domain: string[];      //owl:Class(es)
  range: 'xsd:string' | 'xsd:boolean' | 'xsd:integer' | 'xsd:double' | 'xsd:date' | 'xsd:dateTime';
  multilingual?: boolean; // поддержка языковых тегов
}
```

**Поддерживаемые XSD-типы:**
- `xsd:string` — с многоязычностью
- `xsd:boolean`
- `xsd:integer`
- `xsd:double`
- `xsd:date`
- `xsd:dateTime`

### 4. Аннотации

**Стандартные аннотации:**

| Предикат | Кратность | Назначение |
|-----------|--------------|---------|
| `rdfs:label` | 1..n (по языкам) | Наименование |
| `rdfs:comment` | 0..n | Описание |
| `rdfs:seeAlso` | 0..n | См. также |
| `owl:versionInfo` | 0..1 | Версия |
| `vedo:deprecated` | 0..1 | Устаревший |
| `vedo:archive` | 0..1 | Мягкое удаление |

**Поддержка многоязычности:**
- Языковые теги: `ru`, `en`, `de` и т. д.
- Формат: `{"ru": "Текст", "en": "Text"}`
- UI: переключатель языка контента

### 5. Иерархия Class (дерево)

**Возможности:**
- Drag-n-drop для изменения `rdfs:subClassOf`
- Поиск по дереву (реальное время)
- Фильтрация по типу (abstract, deprecated)
- Правая панель: унаследованные свойства от всех родителей

### 6. Обработка ошибок

**Дублирующийся ID:**
```
Error: "Entity with such ID already exists"
Resolution: Изменить наименование (автогенерация нового ID)
```

**Циклическое наследование:**
```
SparQL check: ASK { ?class rdfs:subClassOf+ ?class . }
Error: "Cyclic inheritance detected"
```

**Удаление при наличии ссылок:**
```
API: DELETE with VerifyReference=1
Error: "Object {ref} refers to {class}"
```

**Стратегии удаления:**
- **Soft delete**: `vedo:archive = true` (класс помечается как устаревший)
- **Hard delete**: Удалить класс и все его потомков (требует подтверждения)

### 7. Правила защиты

1. **Неизменяемость ID**: Нельзя изменить после создания
2. **Проверка ссылок**: Проверка ссылок перед удалением
3. **Проверка объектов**: Нельзя удалить класс с индивидами (без подтверждения)
4. **Предупреждение о каскаде**: Предупреждение при удалении класса с потомками

## События

| Событие | Триггер | Действие |
|-------|---------|--------|
| `ClassCreated` | Save в форме класса | Создать узел в Neo4j, сохранить коммит |
| `ClassUpdated` | Изменение полей | Обновить в Neo4j, сохранить коммит |
| `ClassDeleted` | Подтверждение удаления | Мягкое или жёсткое удаление |
| `ClassHierarchyChanged` | Drag-n-drop | Обновить `rdfs:subClassOf` |
| `PropertyCreated` | Save в форме свойства | Создать свойство в графе |
| `PropertyUpdated` | Изменение domain/range | Обновить в графе |

## Примеры API

**Создание класса:**
```json
POST /api/v1/ontologies/{ontologyId}/classes
{
  "id": "ТепловаяЭлектростанция",
  "label": {"ru": "Тепловая электростанция", "en": "Thermal power plant"},
  "comment": {"ru": ["Вырабатывает электроэнергию из тепла"]},
  "parents": ["ЭнергетическийОбъект"]
}
```

**Ответ:**
```json
{
  "id": "ТепловаяЭлектростанция",
  "status": "created",
  "version": "v2"
}
```

## Зависимости

- **Neo4j**: Хранение TBox (классы, свойства, иерархия)
- **PostgreSQL**: Метаданные (createdBy, createdAt, version)
- **Redis**: Блокировки при редактировании
- **Versioning Service**: Сохранение коммитов при изменениях

## Проектные решения

| Вопрос | Решение |
|--------|---------|
| `owl:equivalentClass` | Поддержка через UI (future) |
| `owl:Restriction` | Через API (advanced) |
| SHACL валидация | На сервере (клиент — опционально) |
