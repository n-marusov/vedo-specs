# Сервис метрик — Управление метриками и аналитика

| Атрибут | Значение |
|---------|----------|
| **ID** | REQ-NFR.OPS.metrics |
| **Уровень** | NFR |
| **Атрибут качества** | Supportability |
| **Приоритет** | P0 |
| **Статус** | УТВЕРЖДЕНО |
| **Источник** | — |
| **Критерии приёмки** | — |

---

## Обзор

Сервис метрик (Python) собирает и предоставляет аналитику по онтологиям: количество классов, свойств, индивидов, глубина иерархии, время выполнения запросов.

## Ключевые возможности

### 1. Метрики онтологии

```typescript
interface OntologyMetrics {
  ontologyId: string;
  
  // Counts
  classCount: number;
  objectPropertyCount: number;
  datatypePropertyCount: number;
  individualCount: number;
  axiomCount: number;        // Все триплеты
  
  // Hierarchy
  maxClassDepth: number;     // Максимальная глубина в дереве классов
  avgClassChildren: number;   // Среднее количество детей на класс
  
  // Complexity
  branchingFactor: number;   // Коэффициент ветвления
  orphanClasses: number;     // Классы без родителей (кроме owl:Thing)
  deepClasses: number;       // Классы с глубиной > 5
  
  // Data coverage
  individualsPerClass: number;  // Среднее количество индивидов на класс
  propertyUsageRate: number;    // Процент используемых свойств
}
```

### 2. Метрики производительности

```typescript
interface QueryMetrics {
  queryId: string;
  timestamp: Date;
  
  // Timing
  durationMs: number;
  parseDurationMs: number;
  executionDurationMs: number;
  
  // Resource usage
  neo4jReadTimeMs: number;
  cacheHit: boolean;
  resultSize: number;          // Количество возвращённых элементов
  
  // Context
  userId: string;
  ontologyId: string;
  queryType: 'class' | 'property' | 'individual' | 'search';
}
```

### 3. Метрики активности пользователей

```typescript
interface UserActivityMetrics {
  userId: string;
  period: 'day' | 'week' | 'month';
  
  // Activity
  loginCount: number;
  activeMinutes: number;
  operationsCount: number;     // CRUD операции
  
  // Operations breakdown
  classesCreated: number;
  classesModified: number;
  propertiesCreated: number;
  individualsCreated: number;
  
  // Commits
  commitsCount: number;
  mergeCount: number;
}
```

### 4. Метрики состояния системы

```typescript
interface SystemHealth {
  timestamp: Date;
  
  // Services
  services: {
    name: string;
    status: 'healthy' | 'degraded' | 'down';
    responseTimeMs: number;
    errorRate: number;         // Процент ошибок
  }[];
  
  // Database
  neo4j: {
    activeConnections: number;
    queryDurationP95: number;
    cacheHitRatio: number;
  };
  
  redis: {
    memoryUsedPercent: number;
    evictionsPerMinute: number;
    hitRatio: number;
  };
  
  // Queue
  rabbitmq: {
    messagesPending: number;
    consumerCount: number;
  };
}
```

## API Endpoints Метрик

### Metrics API

```
GET    /api/v1/ontologies/{id}/metrics
  → OntologyMetrics для конкретной онтологии

GET    /api/v1/ontologies/{id}/metrics/history
  Query: ?period=day|week|month&from=&to=
  → История метрик за период

GET    /api/v1/metrics/system
  → SystemHealth (все сервисы)

GET    /api/v1/metrics/performance
  Query: ?ontologyId=&userId=&from=&to=
  → Список QueryMetrics с фильтрами

GET    /api/v1/users/{id}/activity
  Query: ?period=day|week|month
  → UserActivityMetrics
```

### Данные дашборда

```
GET    /api/v1/dashboard/overview
  → {
      totalOntologies: number,
      totalClasses: number,
      totalIndividuals: number,
      topUsers: { userId, activity }[],
      recentActivity: Activity[]
    }

GET    /api/v1/dashboard/ontology/{id}
  → Полная аналитика для онтологии
```

## Сбор метрик

### Сбор событий

```python
# Python: metrics service
import asyncio
from aio_pika import Message

class MetricsCollector:
    async def collect_query_metrics(self, query_event: QueryEvent):
        metric = QueryMetrics(
            queryId=query_event.id,
            timestamp=datetime.utcnow(),
            durationMs=query_event.duration,
            userId=query_event.user_id,
            ontologyId=query_event.ontology_id,
            # ...
        )
        await self.db.insert_metric(metric)
        
    async def collect_ontology_metrics(self, ontology_id: str):
        # Запрос к Neo4j для подсчёта
        result = await self.neo4j.execute("""
            MATCH (c:Class)
            OPTIONAL MATCH (c)-[:SUBCLASS_OF]->(p)
            RETURN count(c) as classCount, 
                   avg(size((c)-[:SUBCLASS_OF]->())) as avgChildren
        """)
        # Сохранить в PostgreSQL
```

### Интеграция с Prometheus

```yaml
# Prometheus scrape config
scrape_configs:
  - job_name: 'vedo-metrics'
    static_configs:
      - targets: ['metrics-service:9090']
    metrics_path: '/metrics'
```

**Ключевые метрики для Prometheus:**

```prometheus
# Ontology metrics
vedo_ontology_classes_total{ontology_id="..."} 1500
vedo_ontology_individuals_total{ontology_id="..."} 5000
vedo_ontology_depth_max{ontology_id="..."} 7

# Query performance
vedo_query_duration_seconds{quantile="0.95"} 0.12
vedo_query_cache_hits_total 85000
vedo_query_cache_misses_total 15000

# User activity
vedo_user_operations_total{user_id="...", operation="create"} 245
```

## Дашборд Grafana

### Панели дашборда

1. **Обзор системы**
   - Статус здоровья сервисов (зелёный/красный/жёлтый)
   - Активные соединения (Neo4j, Redis)
   - Частота запросов по сервисам

2. **Производительность запросов**
   - Тепловая карта задержек p50/p95/p99
   - Индикатор доли попаданий в кэш
   - Таблица медленных запросов

3. **Аналитика онтологий**
   - Количество классов/свойств/индивидов (накопительная диаграмма с областями)
   - Распределение глубины иерархии
   - Коэффициент покрытия данными

4. **Активность пользователей**
   - Временная шкала активных пользователей
   - Разбивка операций (круговая диаграмма)
   - Таблица ведущих участников

## Зависимости

- **PostgreSQL**: Хранение временных рядов метрик (TimescaleDB extension)
- **Redis**: Временный кэш агрегированных метрик
- **RabbitMQ**: Очередь для асинхронного сбора метрик
- **Prometheus**: Сбор метрик из всех сервисов
- **Grafana**: Визуализация

## Целевые показатели производительности

| Метрика | Цель |
|--------|--------|
| Ответ Metrics API | < 100 ms |
| Агрегация метрик (1M axioms) | < 5 sec |
| Загрузка дашборда | < 1 sec |
| Задержка обработки событий | < 50 ms |
| Хранение данных (детальные) | 30 days |
| Хранение данных (агрегированные) | 1 year |
