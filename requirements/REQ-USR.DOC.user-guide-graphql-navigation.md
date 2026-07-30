# Руководство пользователя: навигация по онтологии через GraphQL

| Атрибут | Значение |
|---------|----------|
| **ID** | REQ-USR.DOC.user-guide-graphql-navigation |
| **Уровень** | USR |
| **Атрибут качества** | Usability |
| **Приоритет** | P1 |
| **Статус** | УТВЕРЖДЕНО |
| **Источник** | — |
| **Критерии приёмки** | — |

---


## Назначение

GraphQL API используется разработчиками frontend и интеграционных UI-компонентов для навигации по графу онтологии: загрузки начальной сцены, раскрытия узлов, подгрузки дочерних классов, входящих и исходящих связей, значений свойств и деталей узла.

GraphQL предназначен для типовых навигационных сценариев 2D/3D интерфейса. SPARQL остаётся отдельным API для аналитики: сложных фильтров, агрегаций и обходов графа, которые не выражаются навигационной GraphQL-схемой.

## Базовая модель

GraphQL-схема не генерирует отдельное поле для каждого пользовательского свойства онтологии. Вместо этого используются стабильные базовые типы:

- `Entity` — общий интерфейс для сущностей графа.
- `Class` — класс онтологии.
- `Individual` — индивид или экземпляр класса.
- `Property` — свойство или отношение.

Динамические свойства и связи возвращаются через контейнеры:

- `propertyValues` — значения свойств текущей сущности.
- `outgoingEdges(relationType: String)` — исходящие связи, опционально отфильтрованные по типу отношения.
- `incomingEdges(relationType: String)` — входящие связи, опционально отфильтрованные по типу отношения.

## Пагинация

Все списки в GraphQL-навигации используют cursor pagination. Клиент передаёт:

- `first` — сколько элементов запросить.
- `after` — непрозрачный курсор, с которого продолжить чтение.

Ответ содержит:

- `endCursor` — курсор последнего элемента текущей страницы.
- `hasNextPage` — есть ли следующая страница.

Пример:

```graphql
query GetClassChildren($classId: ID!, $after: String) {
  class(id: $classId) {
    id
    label
    children(first: 20, after: $after) {
      edges {
        cursor
        node { id label type }
      }
      pageInfo { endCursor hasNextPage }
    }
  }
}
```

Если `hasNextPage = true`, клиент передаёт `endCursor` в следующем запросе как `after`.

## Значения свойств

Значения пользовательских свойств возвращаются через типизированные поля на типе `Individual`:

- `literalValues` — свойства-литералы (DatatypeProperty): строка, число, дата.
- `referenceValues` — ссылочные свойства (ObjectProperty): связь с другим индивидом.

```graphql
query GetIndividualProperties($ontologyId: ID!, $individualId: ID!) {
  individual(ontologyId: $ontologyId, individualId: $individualId) {
    id
    label
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

Такой формат нужен потому, что пользователи создают собственные свойства, неизвестные на момент компиляции GraphQL-схемы — они не регистрируются как статические поля, а возвращаются через контейнеры `literalValues` и `referenceValues`.

## Графовая окрестность для 2D/3D-навигатора

Для 2D/3D-сцены клиент запрашивает фокусный узел и ограниченную глубину обхода через запрос `graphNeighborhood`:

```graphql
query GetNeighborhood($ontologyId: ID!, $classId: ID!, $depth: Int = 2) {
  graphNeighborhood(ontologyId: $ontologyId, classId: $classId, depth: $depth) {
    nodes { id label entityType }
    edges { sourceId targetId propertyId propertyLabel }
  }
}
```

Клиент должен начинать с небольшой глубины и дозагружать соседние связи при клике, раскрытии узла или прокрутке списка. `graphNeighborhood` возвращает все узлы и рёбра одной порцией (в пределах `depth`), без курсорной пагинации — для визуализации это эффективнее, чем постраничная подгрузка.

## Ограничения

GraphQL endpoint применяет ограничения для защиты Ontology Service и хранилища графа:

- `depth limit` — максимальная вложенность GraphQL-запроса.
- `query complexity` — суммарная стоимость запроса по весам полей, глубине и размеру страниц.
- `timeout` — максимальное время выполнения запроса.
- `result cap` — максимальное количество узлов или связей в одном ответе.
- `rate limiting` — ограничение количества запросов в минуту на пользователя.

Если запрос отклонён, клиент должен показать пользователю понятную рекомендацию: уменьшить глубину, сузить типы связей, уменьшить `first` или продолжить загрузку по курсору.

## Introspection и Playground

Разработчики используют GraphQL introspection и GraphQL Playground для экспериментов со схемой, доступными типами и примерами запросов.

Точный URL Playground зависит от окружения. Ожидаемый путь для dev/staging: `/graphql/playground`. В production Playground может быть отключён политикой безопасности, но introspection для авторизованных разработчиков должна быть доступна в согласованном режиме.

## Открытые вопросы

- Точная production-политика для GraphQL Playground и introspection должна быть согласована с командой безопасности.
- Конкретные значения лимитов `depth limit`, `query complexity`, `first` и rate limiting могут отличаться по окружениям и должны быть вынесены в конфигурацию.
