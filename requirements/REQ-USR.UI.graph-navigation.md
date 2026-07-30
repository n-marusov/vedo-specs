# F1: Просмотр графа и навигация

| Атрибут | Значение |
|---------|----------|
| **ID** | REQ-USR.UI.graph-navigation |
| **Уровень** | USR |
| **Атрибут качества** | Usability |
| **Приоритет** | P0 |
| **Статус** | УТВЕРЖДЕНО |
| **Источник** | — |
| **Критерии приёмки** | — |

---


## Обзор

Визуализация и навигация по онтологии: интерактивное дерево классов, поиск, фильтрация, просмотр связей и графа.

Для навигации по графу используется GraphQL API: 2D/3D визуализация, пагинированные списки, подгрузка порций графа и обход связей. SPARQL остаётся отдельным интерфейсом для сложных аналитических запросов, не покрываемых навигационной схемой.

## Функциональные требования

### REQ-FR-GRAPH-NAV-0001 Навигация по графу

Система должна предоставлять API для навигации по онтологии, поддерживающее получение классов, свойств, индивидов и связей между ними порциями.

### REQ-FR-GRAPH-NAV-0002 Динамические свойства

API должен позволять получать значения свойств для произвольных классов и индивидов без необходимости заранее описывать каждое пользовательское свойство в схеме API. Для этого используются контейнеры `propertyValues` и `outgoingEdges`/`incomingEdges`.

### REQ-FR-GRAPH-NAV-0003 Обход связей произвольного типа

API должен поддерживать навигацию по исходящим и входящим связям произвольного типа. Клиент может передать ID или тип связи, если хочет ограничить результат конкретным отношением.

### REQ-FR-GRAPH-NAV-0004 Пагинация на основе курсоров

Все операции получения списков должны поддерживать пагинацию на основе курсоров: `first`, `after`, `endCursor`, `hasNextPage`. Offset-пагинация не используется для навигационных списков, потому что она нестабильна при параллельных изменениях графа.

### REQ-FR-GRAPH-NAV-0005 Аналитические запросы

Система должна предоставлять отдельный API для аналитических запросов к онтологии, включая сложные фильтры, агрегации и обход графа переменной глубины. Такие сценарии выполняются через SPARQL, а не через GraphQL-навигацию.

## Нефункциональные требования

### REQ-NFR-GRAPH-NAV-SEC-0001 Ограничение глубины

Система должна ограничивать максимальную глубину GraphQL-навигационного запроса для предотвращения рекурсивных или чрезмерно вложенных запросов.

### REQ-NFR-GRAPH-NAV-SEC-0002 Сложность запроса

Система должна оценивать сложность GraphQL-запроса по весам полей, вложенности и размеру запрашиваемых страниц. Запросы выше установленного порога отклоняются до обращения к хранилищу.

### REQ-NFR-GRAPH-NAV-PERF-0001 Тайм-аут выполнения

Навигационные и аналитические запросы должны иметь ограничение по времени выполнения.

### REQ-NFR-GRAPH-NAV-PERF-0002 Лимит количества данных

API должен ограничивать максимальное количество элементов, возвращаемых в одном ответе, и максимальное значение `first` для connection-полей.

### REQ-NFR-GRAPH-NAV-SEC-0003 Ограничение частоты запросов

Система должна применять ограничение частоты запросов для каждого пользователя, чтобы защитить Ontology Service и хранилище графа от перегрузки.

## Пользовательские истории

1. **Инженер знаний** открывает онтологию и видит дерево классов
2. **Аналитик данных** ищет класс по имени и изучает его свойства
3. **ИТ-архитектор** визуализирует связи между классами

## Компоненты

### 1. Дерево классов

**Возможности:**
- Иерархическое отображение `rdfs:subClassOf`
- Виртуализация списка (для онтологий с 1000+ классами)
- Drag-n-drop для изменения иерархии
- Контекстное меню (создать, редактировать, удалить)
- Иконки по типу: Class, ObjectProperty, DatatypeProperty, Individual

**Поиск и фильтрация:**
- Фильтрация в реальном времени по мере ввода пользователем
- Поиск по: label, comment, ID
- Фильтр по: abstract/deprecated, has-individuals, language

**Состояние:**
```typescript
interface ClassTreeState {
  rootClasses: string[];           // Классы без родителей
  expandedNodes: Set<string>;      // Развёрнутые узлы
  selectedNode: string | null;     // Выбранный класс
  filter: {
    query: string;
    showAbstract: boolean;
    showDeprecated: boolean;
  };
}
```

### 2. Панель деталей узла

**Появляется, когда пользователь выбирает класс или свойство в дереве**

**Для Class:**
- Базовая информация: id, label (по языкам), comment
- Parents: список суперклассов (кликабельные)
- Children: список подклассов (кликабельные)
- Properties: унаследованные от родителей (read-only, серые) и определённые в этом классе (редактируемые)
- Individuals: список экземпляров (количество + предпросмотр)

**Для ObjectProperty:**
- Domain: к какому классу принадлежит
- Range: тип значения
- Characteristics: transitive, symmetric, functional
- Inverse property

**Для DatatypeProperty:**
- Domain, Range (XSD type)
- Индикатор поддержки многоязычности

### 3. Визуализация графа

**Технология:** @vue-flow/core (2D) + опционально Three.js (3D)

**Возможности (2D):**
- Показ выбранного класса + его родители + потомки (3 уровня)
- Автоматическая раскладка (force-directed или dagre)
- Управление zoom/pan
- Отображение типов связей: `rdfs:subClassOf`, `rdfs:domain`, `rdfs:range`
- Подсветка при hover

**Режимы просмотра:**
- **Tree**: древовидная раскладка (по умолчанию)
- **Force**: физическая симуляция (для сложных связей)
- **Radial**: центрирование на выбранном узле
- **3D**: трёхмерная визуализация графа (WebGL/Three.js)

### 4. 3D-визуализация графа (опционально для MVP, WebGL)

**Назначение:** Погружение в структуру онтологии, визуализация сложных иерархий и связей

**Технологический стек:**
- **Three.js** для WebGL рендеринга
- **force-graph** или **3d-force-graph** для 3D layout
- **@react-three/fiber** (адаптер для Vue) или нативный Three.js

**Возможности:**

| Возможность | Описание |
|---------|-------------|
| **Орбитальная камера** | Свободное вращение, zoom, pan в 3D-пространстве |
| **Force-directed layout** | Автоматическая раскладка узлов в 3D |
| **Размер узла** | Пропорционально количеству связей или потомков |
| **Цветовое кодирование** | По типу: Class (синий), ObjectProperty (зелёный), DatatypeProperty (жёлтый), Individual (серый) |
| **Метки** | Отображение label при hover или в режиме always-visible |
| **Типы рёбер** | Цвет и толщина по типу связи |
| **Выбор** | Клик на узел → подсветка связей + детали в панели |
| **Фильтрация** | Скрытие по типу, глубине, label |

**Пользовательские взаимодействия:**

```
Mouse:
- Left drag: orbit camera (вращение вокруг графа)
- Right drag: pan (перемещение камеры)
- Scroll: zoom in/out
- Click node: select + show details
- Double-click: focus on node

Touch (mobile):
- One finger drag: orbit
- Two finger pinch: zoom
- Two finger drag: pan
- Tap: select node
```

**Производительность (3D):**

| Метрика | Цель |
|--------|--------|
| Максимум отрисованных узлов | 500-1000 |
| Частота кадров | 60 FPS |
| Начальная загрузка (500 nodes) | < 2 sec |
| Вычисление layout | < 1 sec (WebWorker) |

**Оптимизации рендеринга:**
- **Instanced rendering** для повторяющихся геометрий
- **LOD** (Level of Detail): дальние узлы = простые сферы
- **Frustum culling**: не рендерить невидимые узлы
- **WebWorker**: layout computation отдельно от main thread
- **GPU instancing** для рёбер (линий)

**Алгоритмы раскладки (3D):**

```typescript
type Layout3D = 
  | 'force'           // Force-directed (default)
  | 'circular'        // Ring layout
  | 'hierarchical'    // Tree in 3D space
  | 'sphere'          // Spherical packing
  | 'custom';         // User-defined via API
```

**UI-элементы управления (3D-панель):**

```
[3D View] [Reset Camera] [Layout: Force ▼] [Depth: 5] [Show: All ▼]
[Filters ☑ Classes ☑ Properties ☐ Individuals]
[Animate: ON] [Labels: On Hover ▼]
```

**Доступность:**
- Screen reader озвучивает выбранный узел
- Навигация с клавиатуры (Tab, стрелки, Enter)
- Опция палитры, удобной для людей с нарушением цветовосприятия

**Архитектура:**

```
GraphView
├── GraphCanvas2D (@vue-flow)
│   └── Used for: Tree, Force, Radial modes
└── GraphCanvas3D (Three.js)
    ├── SceneManager (camera, lights, renderer)
    ├── GraphData (nodes, edges, layout)
    ├── LayoutEngine (WebWorker)
    └── InteractionHandler (raycasting, events)

State:
- Apollo: graphStore (nodes, edges, selectedNode, layout3D)
- Composables: useThreeScene, useGraphLayout
```

**Progressive enhancement:**
- 2D визуализация — базовая (в MVP)
- 3D визуализация — включается по клику на кнопку
- WebGL fallback: если WebGL не поддерживается → только 2D

## Цветовая схема (АрхиГраф)

| Тип узла | Цвет | Пример |
|----------|------|--------|
| Class | `#4CAF50` (зелёный) | Основные узлы графа |
| ObjectProperty | `#2196F3` (синий) | Связи между классами |
| DatatypeProperty | `#FF9800` (оранжевый) | Атрибуты |
| Individual | `#9C27B0` (фиолетовый) | Экземпляры |

## Целевые показатели производительности

| Режим | Размер графа | FPS |
|-------|-------------|-----|
| 2D | < 1000 nodes | 60 |
| 2D | 1000-5000 | 30-40 |
| 2D | 5000+ | 15-20 (с виртуализацией) |
| 3D | < 500 nodes | 60 |
| 3D | 500-2000 | 30 |
| 3D | 2000-5000 | 20 (LOD) |

## Экспорт

- **PNG/SVG**: Скриншот текущего вида
- **GLTF/GLB**: Полный 3D граф (binary, Draco compression)
- **Turtle**: Только данные (без layout)

## Решения по UI-дизайну

Эти вопросы решаются на уровне UI дизайна:
- Blank nodes отображаются как узлы без label (tech debt, не показываем в MVP)
- Глубина графа по умолчанию: 3 уровня (настраивается в настройках)
- Layout сохраняется в localStorage пользователя

## Анимация (realtime)

- **Создание узла**: Пульсация + fade in
- **Удаление узла**: Fade out
- **Создание ребра**: Рост линии
- **Отключить для**: > 1000 nodes (performance)

## Навигационная цепочка

**Формат:** `Онтология > Класс > Подкласс`

**Возможности:**
- Кликабельные сегменты пути
- История назад/вперёд

## API-эндпоинты

### REST API (внешний фасад API Gateway)

Эти маршруты публикуются только API Gateway. Ontology Service не должен реализовывать функциональные REST-хендлеры для навигации по графу; API Gateway преобразует публичные REST/GraphQL запросы в gRPC/protobuf вызовы Ontology Service.

```
GET /api/v1/ontologies/{ontologyId}/classes/tree
→ Returns hierarchical tree structure

GET /api/v1/ontologies/{ontologyId}/classes/{classId}
→ Returns class details with parents, children, properties

GET /api/v1/ontologies/{ontologyId}/search?q={query}&type={class|property}
→ Search with highlighting
```

### GraphQL API (frontend)

Для фронтенда используется **GraphQL** (через Apollo Client) для получения фрагментов онтологии. Это основной API навигации по графу: клиент запрашивает только нужные поля, подгружает порции графа по курсорам и раскрывает связи по мере пользовательского действия.

GraphQL в VEDO Core — **строго read-only навигация по графу онтологии** (11 резолверов). Все мутации, версионирование, орг-модель, SPARQL, метрики и метаданные онтологии вынесены в REST. Подробнее: ADR-DES.API.graphql-sparql-split-strategy.md, ADR-DES.API.rest-graphql-mutation-boundary.md.

Схема не пытается статически описать каждое пользовательское свойство онтологии. Вместо этого фиксируются базовые типы `Entity`, `Class`, `Individual`, `Property`, а значения пользовательских свойств возвращаются через поля `literalValues` (свойства-литералы) и `referenceValues` (ссылочные свойства) на типе `Individual`. Навигация по графу выполняется через специализированные запросы: `classTree` (иерархия), `classAncestors` (хлебные крошки), `classDescendants` (поддерево), `graphNeighborhood` (соседние узлы и рёбра).

Все списки используют **Relay Cursor Connections** — пагинацию на основе курсоров (`first`, `after`, `edges`, `pageInfo`, `totalCount`). Offset-пагинация не используется (см. REQ-FR-GRAPH-NAV-0004).

**Схема:**

```graphql
# ═══════════════════════════════════════════════════════════════════
# VEDO Core — GraphQL: только навигация по графу онтологии
# ═══════════════════════════════════════════════════════════════════

# ── Entity interface ──────────────────────────────────────────────

interface Entity {
  id: ID!
  label: String!
  comment: String
  entityType: EntityType!
}

enum EntityType {
  CLASS
  INDIVIDUAL
  PROPERTY
}

# ── Class ─────────────────────────────────────────────────────────

type Class implements Entity {
  id: ID!
  label: String!
  comment: String
  entityType: EntityType!
  parents: [String!]!
  children: [String!]!
  isAbstract: Boolean!
  isDeprecated: Boolean!
}

type ClassSummary {
  id: ID!
  label: String!
  comment: String
  parents: [String!]!
}

type ClassTreeNode {
  id: ID!
  label: String!
  comment: String
  children: [ClassTreeNode!]!
}

type BreadcrumbItem {
  id: ID!
  label: String!
}

# ── Property ──────────────────────────────────────────────────────

type Property implements Entity {
  id: ID!
  label: String!
  comment: String
  entityType: EntityType!
  propertyType: PropertyType!
  domains: [String!]!
  ranges: [String!]!
  xsdType: String
  characteristics: PropertyCharacteristics!
  annotations: [Annotation!]!
}

enum PropertyType {
  OBJECT
  DATATYPE
  ANNOTATION
}

type PropertyCharacteristics {
  functional: Boolean!
  inverseFunctional: Boolean!
  transitive: Boolean!
  symmetric: Boolean!
}

type Annotation {
  propertyIri: String!
  value: String!
}

type PropertySummary {
  id: ID!
  label: String!
  propertyType: PropertyType!
  xsdType: String
  domains: [String!]!
}

# ── Individual ────────────────────────────────────────────────────

type Individual implements Entity {
  id: ID!
  label: String!
  comment: String
  entityType: EntityType!
  classId: String!
  classLabel: String!
  literalValues: [LiteralValue!]!
  referenceValues: [ReferenceValue!]!
}

type IndividualSummary {
  id: ID!
  label: String!
  comment: String
  classId: String!
  classLabel: String!
}

type LiteralValue {
  propertyId: String!
  propertyLabel: String!
  value: String!
  xsdType: String
  valueId: String
}

type ReferenceValue {
  propertyId: String!
  propertyLabel: String!
  targetId: String!
  targetLabel: String!
  edgeId: String
}

# ── Connections (Relay Cursor) ────────────────────────────────────

type PageInfo {
  startCursor: String
  endCursor: String
  hasNextPage: Boolean!
  hasPreviousPage: Boolean!
}

type ClassConnection {
  edges: [ClassEdge!]!
  pageInfo: PageInfo!
  totalCount: Int!
}

type ClassEdge {
  cursor: String!
  node: ClassSummary!
}

type PropertyConnection {
  edges: [PropertyEdge!]!
  pageInfo: PageInfo!
  totalCount: Int!
}

type PropertyEdge {
  cursor: String!
  node: PropertySummary!
}

type IndividualConnection {
  edges: [IndividualEdge!]!
  pageInfo: PageInfo!
  totalCount: Int!
}

type IndividualEdge {
  cursor: String!
  node: IndividualSummary!
}

# ── Graph Neighborhood ────────────────────────────────────────────

type GraphNeighborhood {
  nodes: [GraphNode!]!
  edges: [GraphEdge!]!
}

type GraphNode {
  id: ID!
  label: String!
  entityType: EntityType!
}

type GraphEdge {
  sourceId: ID!
  targetId: ID!
  propertyId: ID!
  propertyLabel: String!
}

# ── Query Root (11 резолверов) ────────────────────────────────────

type Query {
  # Точечные lookup-ы
  class(ontologyId: ID!, classId: ID!): Class
  property(ontologyId: ID!, propertyId: ID!): Property
  individual(ontologyId: ID!, individualId: ID!): Individual

  # Списки — cursor pagination (Relay Connection)
  classes(
    ontologyId: ID!
    q: String
    first: Int = 20
    after: String
  ): ClassConnection!

  properties(
    ontologyId: ID!
    q: String
    propertyType: PropertyType
    first: Int = 20
    after: String
  ): PropertyConnection!

  individuals(
    ontologyId: ID!
    classId: ID!
    q: String
    first: Int = 20
    after: String
  ): IndividualConnection!

  # Иерархия классов
  classTree(ontologyId: ID!): [ClassTreeNode!]!
  classAncestors(ontologyId: ID!, classId: ID!): [BreadcrumbItem!]!
  classDescendants(
    ontologyId: ID!
    classId: ID!
    maxDepth: Int = 10
  ): [ClassTreeNode!]!

  # Визуализация графа
  graphNeighborhood(
    ontologyId: ID!
    classId: ID!
    depth: Int = 2
  ): GraphNeighborhood!

  # Поиск
  autocompleteClasses(
    ontologyId: ID!
    q: String!
    limit: Int = 20
  ): [ClassSummary!]!
}
```

**Пример: получение класса с иерархией и neighbourhood:**

```graphql
query GetClassContext($ontologyId: ID!, $classId: ID!) {
  class(ontologyId: $ontologyId, classId: $classId) {
    id
    label
    entityType
    isAbstract
    isDeprecated
    parents
    children
  }
  classAncestors(ontologyId: $ontologyId, classId: $classId) {
    id
    label
  }
  graphNeighborhood(ontologyId: $ontologyId, classId: $classId, depth: 2) {
    nodes { id label entityType }
    edges { sourceId targetId propertyId propertyLabel }
  }
}
```

**Пример: список классов с курсорной пагинацией:**

```graphql
query ListClasses($ontologyId: ID!, $q: String, $after: String) {
  classes(ontologyId: $ontologyId, q: $q, first: 20, after: $after) {
    edges {
      cursor
      node { id label parents }
    }
    pageInfo { endCursor hasNextPage }
    totalCount
  }
}
```

**Пример: индивид с полными значениями свойств:**

```graphql
query GetIndividual($ontologyId: ID!, $individualId: ID!) {
  individual(ontologyId: $ontologyId, individualId: $individualId) {
    id
    label
    classId
    classLabel
    literalValues {
      propertyId
      propertyLabel
      value
      xsdType
    }
    referenceValues {
      propertyId
      propertyLabel
      targetId
      targetLabel
    }
  }
}
```

## Поток данных (GraphQL)

```
User selects class in ClassTree
  → Apollo Client sends classTree / classDescendants query
  → API Gateway routes to Ontology Service (Rust)
  → Cypher query to Neo4j
  → Response: ClassTreeNode[] with nested children
  → Apollo cache updated
  → Components re-render with reactive data

User clicks node in graph visualization
  → Apollo Client sends graphNeighborhood query (depth, classId)
  → Ontology Service traverses Neo4j graph
  → Response: GraphNeighborhood { nodes[], edges[] }
  → Vue Flow / Three.js renders nodes and edges

User opens individual details
  → Apollo Client sends individual query with literalValues, referenceValues
  → Ontology Service fetches property values from Neo4j
  → Response: Individual with full property assignments
  → DetailPanel renders property table
```

## Стратегия кэширования

- **Apollo InMemoryCache**: автоматическое кэширование по id
- **Redis**: кэш для структуры дерева (TTL 60s)
- **ETag**: для деталей класса
- **Prefetch**: загрузка соседних узлов при hover

## Иерархия компонентов

```
GraphView (page)
├── ClassTree
│   ├── VirtualList (vue-virtual-scroller)
│   ├── SearchBar
│   └── TreeNode
├── DetailPanel
│   ├── ClassInfo (GraphQL: class(id))
│   ├── ParentsList
│   ├── ChildrenList
│   ├── PropertiesTable
│   └── IndividualsPreview
├── GraphCanvas2D (@vue-flow)
│   └── GraphNodes (loaded via graphNeighborhood query)
├── GraphCanvas3D (Three.js)
└── Breadcrumb
```

## Зависимости

- **Neo4j**: обход графа для parent/child-запросов
- **Redis**: кэш для структуры дерева
- **Three.js**: 3D-рендеринг (опционально, progressive enhancement)
- **@vue-flow/core**: 2D-визуализация графа
