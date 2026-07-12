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

Схема не пытается статически описать каждое пользовательское свойство онтологии. Вместо этого она фиксирует базовые интерфейсы `Entity`, `Class`, `Individual`, `Property`, а динамические свойства и связи возвращает через контейнеры `propertyValues`, `outgoingEdges` и `incomingEdges`.

**Схема:**

```graphql
interface Entity {
  id: ID!
  label(lang: String): String!
  type: EntityType!
}

enum EntityType {
  CLASS
  INDIVIDUAL
  PROPERTY
}

type Ontology {
  id: ID!
  name: String!
  rootClasses(first: Int = 50, after: String): ClassConnection!
  classes(first: Int = 50, after: String): ClassConnection!
  properties(first: Int = 50, after: String): PropertyConnection!
}

type Class implements Entity {
  id: ID!
  label(lang: String): String!
  comment(lang: String): String
  type: EntityType!
  parents(first: Int = 50, after: String): ClassConnection!
  children(first: Int = 50, after: String): ClassConnection!
  propertyValues(first: Int = 50, after: String): PropertyValueConnection!
  outgoingEdges(relationType: String, first: Int = 50, after: String): GraphEdgeConnection!
  incomingEdges(relationType: String, first: Int = 50, after: String): GraphEdgeConnection!
  isAbstract: Boolean
  isDeprecated: Boolean
}

type Property implements Entity {
  id: ID!
  label(lang: String): String!
  type: EntityType!
  propertyKind: PropertyKind!
  domain: [Class!]!
  range: [Entity!]!
  isMultilingual: Boolean
}

enum PropertyKind {
  OBJECT
  DATATYPE
  ANNOTATION
}

type Individual implements Entity {
  id: ID!
  label(lang: String): String!
  type: EntityType!
  classes(first: Int = 50, after: String): ClassConnection!
  propertyValues(first: Int = 50, after: String): PropertyValueConnection!
  outgoingEdges(relationType: String, first: Int = 50, after: String): GraphEdgeConnection!
  incomingEdges(relationType: String, first: Int = 50, after: String): GraphEdgeConnection!
}

type PropertyValue {
  property: Property!
  values: [PropertyValueItem!]!
}

union PropertyValueItem = LiteralValue | EntityValue

type LiteralValue {
  value: String!
  datatype: String
  language: String
}

type EntityValue {
  entity: Entity!
}

type Query {
  ontology(id: ID!): Ontology
  class(id: ID!): Class
  entity(id: ID!): Entity
  searchOntology(query: String!, first: Int = 20, after: String): EntityConnection!
  subgraph(focusId: ID!, depth: Int = 2, first: Int = 100, after: String): GraphEdgeConnection!
}

type GraphEdge {
  node: Entity!
  predicate: Property!
}

type PageInfo {
  endCursor: String
  hasNextPage: Boolean!
}

type ClassConnection { edges: [ClassEdge!]!, pageInfo: PageInfo! }
type ClassEdge { cursor: String!, node: Class! }
type PropertyConnection { edges: [PropertyEdge!]!, pageInfo: PageInfo! }
type PropertyEdge { cursor: String!, node: Property! }
type EntityConnection { edges: [EntityEdge!]!, pageInfo: PageInfo! }
type EntityEdge { cursor: String!, node: Entity! }
type PropertyValueConnection { edges: [PropertyValueEdge!]!, pageInfo: PageInfo! }
type PropertyValueEdge { cursor: String!, node: PropertyValue! }
type GraphEdgeConnection { edges: [GraphEdgeResult!]!, pageInfo: PageInfo! }
type GraphEdgeResult { cursor: String!, node: GraphEdge! }
```

**Пример запроса для 3D-навигатора:**

```graphql
query GetSubgraph($classId: ID!, $depth: Int = 2, $after: String) {
  class(id: $classId) {
    id
    label
    parents(first: 20) {
      edges { node { id label } }
      pageInfo { endCursor hasNextPage }
    }
    children(first: 20, after: $after) {
      edges { cursor node { id label } }
      pageInfo { endCursor hasNextPage }
    }
    outgoingEdges(relationType: "subClassOf", first: 50) {
      edges {
        node {
          node { id label type }
          predicate { id label }
        }
      }
      pageInfo { endCursor hasNextPage }
    }
  }
  subgraph(focusId: $classId, depth: $depth, first: 100, after: $after) {
    edges {
      cursor
      node {
        node { id label type }
        predicate { id label }
      }
    }
    pageInfo { endCursor hasNextPage }
  }
}
```

**Настройка Apollo Client (Vue 3):**

```typescript
// src/apollo/client.ts
import { ApolloClient, InMemoryCache, createHttpLink } from '@apollo/client/core'

const httpLink = createHttpLink({ uri: '/graphql' })

export const apolloClient = new ApolloClient({
  link: httpLink,
  cache: new InMemoryCache({
    typePolicies: {
      Class: { keyFields: ['id'] },
      Individual: { keyFields: ['id'] },
      Property: { keyFields: ['id'] },
    }
  })
})
```

**Composable для загрузки графа:**

```typescript
// src/composables/useOntologyGraph.ts
import { useQuery, useSubscription } from '@apollo/client/vue3'
import gql from 'graphql-tag'

const CLASS_GRAPH_QUERY = gql`
  query ClassGraph($focusId: ID!, $depth: Int = 2, $after: String) {
    subgraph(focusId: $focusId, depth: $depth, first: 100, after: $after) {
      edges {
        cursor
        node {
          node { id label type }
          predicate { id label }
        }
      }
      pageInfo { endCursor hasNextPage }
    }
  }
`

export function useOntologyGraph(classId: Ref<string>) {
  const { result, loading, error } = useQuery(CLASS_GRAPH_QUERY, () => ({
    focusId: classId.value,
    depth: 3
  }))
  
  return { edges: computed(() => result.value?.subgraph?.edges ?? []),
           pageInfo: computed(() => result.value?.subgraph?.pageInfo),
           loading, error }
}
```

**Обновления в реальном времени (subscriptions):**

```graphql
type Subscription {
  classUpdated(id: ID!): Class!
  ontologyChanged(ontologyId: ID!): OntologyChange!
}
```

## Поток данных (GraphQL)

```
User selects class
  → useOntologyGraph(classId) composable
  → Apollo Client sends GraphQL query with depth, first and after cursor
  → API Gateway routes to Ontology Service (Rust)
  → Cypher query to Neo4j
  → Response: connection edges + pageInfo { endCursor, hasNextPage }
  → Apollo cache updated
  → Components re-render with reactive data
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
│   └── GraphNodes (loaded via classGraph query)
├── GraphCanvas3D (Three.js)
└── Breadcrumb
```

## Зависимости

- **Neo4j**: обход графа для parent/child-запросов
- **Redis**: кэш для структуры дерева
- **Three.js**: 3D-рендеринг (опционально, progressive enhancement)
- **@vue-flow/core**: 2D-визуализация графа
