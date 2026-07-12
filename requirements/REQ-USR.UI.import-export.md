# Import/Export — Импорт и экспорт онтологий

| Атрибут | Значение |
|---------|----------|
| **ID** | REQ-USR.UI.import-export |
| **Уровень** | USR |
| **Атрибут качества** | Functionality |
| **Приоритет** | P1 |
| **Статус** | УТВЕРЖДЕНО |
| **Источник** | — |
| **Критерии приёмки** | — |

---


## Обзор

VEDO Core поддерживает импорт и экспорт онтологий в стандартных форматах RDF: Turtle (.ttl), RDF/XML, OWL (functional syntax). Это обеспечивает совместимость с Protégé, другими редакторами и внешними системами.

## Поддерживаемые форматы

### Входные форматы

| Формат | Расширение | MIME Type | Статус |
|--------|-----------|-----------|--------|
| Turtle | `.ttl` | `application/x-turtle` | ✅ Основной |
| RDF/XML | `.rdf`, `.owl` | `application/rdf+xml` | ✅ |
| N-Triples | `.nt` | `application/n-triples` | ✅ |
| N-Quads | `.nq` | `application/n-quads` | 🔜 Планируется |
| JSON-LD | `.jsonld` | `application/ld+json` | 🔜 Планируется |

### Выходные форматы

| Формат | Расширение | MIME Type | Статус |
|--------|-----------|-----------|--------|
| Turtle | `.ttl` | `application/x-turtle` | ✅ Основной |
| RDF/XML | `.rdf` | `application/rdf+xml` | ✅ |
| OWL Functional | `.owx` | `text/owl-functional` | 🔜 Планируется |
| N-Triples | `.nt` | `application/n-triples` | ✅ |
| JSON-LD | `.jsonld` | `application/ld+json` | 🔜 Планируется |

## Ключевые возможности

### 1. Экспорт

**API endpoints:**

Эти endpoints являются внешним REST-фасадом API Gateway. Ontology Service и Versioning Service должны получать import/export команды только через внутренние gRPC/protobuf вызовы; функциональные REST-хендлеры внутри доменных сервисов запрещены.

```typescript
// Export entire ontology
GET /api/v1/ontologies/{id}/export
  Query: ?format=turtle|rdf-xml|nt&includeImports=true&compression=gzip
  Response: Binary file or streaming

// Export subset (TBox only, ABox only, specific classes)
POST /api/v1/ontologies/{id}/export
  Body: {
    format: 'turtle' | 'rdf-xml' | 'nt',
    filter: {
      type: 'tbox' | 'abox' | 'custom',
      classes?: string[],     // Export only specific classes
      depth?: number,         // Include subclasses to depth N
      includeIndividuals?: boolean
    },
    options: {
      baseUri: string,        // Override base URI
      prefixHeader: boolean,  // Include @prefix declarations
      prettyPrint: boolean,    // Human-readable formatting
    }
  }

// Export commit (specific version)
GET /api/v1/ontologies/{id}/commits/{sha}/export
  Query: ?format=turtle
  Response: Ontology at commit state
```

**Целевые показатели производительности:**
- 100,000 triples → < 10 sec
- Потоковая передача для > 1M triples

### 2. Импорт

**API endpoints:**

Эти endpoints являются внешним REST-фасадом API Gateway. API Gateway оркестрирует Ontology Service и Versioning Service через gRPC/protobuf, включая dry-run, apply, commit и report.

```typescript
// Import from file
POST /api/v1/ontologies/{id}/import
  Content-Type: multipart/form-data
  Body: {
    file: <binary>,            // .ttl, .rdf, .owl
    format: 'turtle' | 'rdf-xml' | 'nt',  // auto-detect if omitted
    mode: 'merge' | 'replace' | 'validate-only',
    options: {
      baseUri: string,         // Override base URI
      createCommit: boolean,   // Create commit after import
      commitMessage: string,
      validate: boolean,       // SHACL validation before import
    }
  }

// Import from URL
POST /api/v1/ontologies/{id}/import
  Body: {
    sourceUrl: string,
    format: 'auto' | 'turtle' | 'rdf-xml',
    mode: 'merge' | 'replace'
  }

// Validate without import
POST /api/v1/ontologies/validate
  Body: {
    content: string,           // RDF content as string
    format: 'turtle' | 'rdf-xml',
    schema: 'owl-dl' | 'shacl' | 'none'
  }
  Response: {
    valid: boolean,
    errors: ValidationError[],
    warnings: ValidationWarning[]
  }
```

### 3. Конвертация форматов

```typescript
// Convert between formats
POST /api/v1/convert
  Body: {
    content: string,
    fromFormat: 'turtle' | 'rdf-xml',
    toFormat: 'turtle' | 'rdf-xml' | 'nt',
    options: {
      baseUri: string,
      prefixHeader: boolean
    }
  }
  Response: {
    content: string,
    fromFormat: string,
    toFormat: string
  }
```

### 4. Потоковый импорт/экспорт

Для больших онтологий (> 100,000 triples) поддерживается потоковая обработка:

```typescript
// Streaming export (chunked response)
GET /api/v1/ontologies/{id}/export/stream
  Query: ?format=turtle&chunkSize=10000
  Response: Server-Sent Events или chunked transfer encoding

// Streaming import
POST /api/v1/ontologies/{id}/import/stream
  Body: Stream of RDF triples
  Response: Progress updates via WebSocket
```

## Каноническая сериализация (Turtle)

Для Git-совместимости используется каноническая сериализация Turtle:

```typescript
interface CanonicalTurtleOptions {
  // Sorting
  sortTriples: 'subject' | 'predicate' | 'none';
  
  // Prefix ordering
  sortPrefixes: boolean;              // Alphabetical
  
  // Blank node handling
  blankNodeMode: 'label' | 'bnode';  // Use blank nodes or :b0, :b1 labels
  
  // Whitespace
  indent: '  ' | '\t' | number;
  newLineAfterTriple: boolean;
  
  // Character encoding
  encoding: 'utf-8' | 'ascii';
}

// Example output
@prefix rdf: <http://www.w3.org/1999/02/22-rdf-syntax-ns#> .
@prefix rdfs: <http://www.w3.org/2000/01/rdf-schema#> .
@prefix owl: <http://www.w3.org/2002/07/owl#> .
@prefix ex: <http://example.org/onto/> .

ex:ТепловаяЭлектростанция
    a owl:Class ;
    rdfs:label "Тепловая электростанция"@ru , "Thermal power plant"@en ;
    rdfs:subClassOf ex:ЭнергетическийОбъект .
```

## Обработка ошибок

### Ошибки импорта

```typescript
interface ImportError {
  code: 'PARSING_ERROR' | 'VALIDATION_ERROR' | 'DUPLICATE_ENTITY' | 'INTEGRITY_ERROR';
  message: string;
  line?: number;           // For parsing errors
  column?: number;
  triples?: number[];      // Affected triples
  details?: {
    expected?: string;
    found?: string;
    context?: string;
  };
}

interface ImportResult {
  success: boolean;
  importedTriples: number;
  importedClasses: number;
  importedProperties: number;
  importedIndividuals: number;
  errors: ImportError[];
  warnings: string[];
  commitId?: string;       // If commit was created
}
```

### Типовые сценарии ошибок

| Ошибка | Причина | Решение |
|-------|-------|------------|
| `PARSING_ERROR: Unexpected token` | Некорректный RDF-синтаксис | Проверьте формат файла, выполните валидацию |
| `VALIDATION_ERROR: Undefined class` | Ссылка на несуществующий класс | Сначала добавьте класс или используйте `mode: 'merge'` |
| `DUPLICATE_ENTITY` | Класс/свойство с тем же ID | Используйте `mode: 'merge'` или переименуйте |
| `INTEGRITY_ERROR: Cyclic inheritance` | Класс наследуется от самого себя | Удалите цикл в источнике |

## UI-компоненты

### Модальное окно экспорта

```
┌──────────────────────────────────────────┐
│ Export Ontology                      ✕   │
├──────────────────────────────────────────┤
│ Format:  [ Turtle (.ttl) ▼ ]              │
│                                          │
│ Options:                                 │
│ [x] Include annotations                  │
│ [ ] Include individuals (ABox)          │
│ [x] Pretty print                         │
│ [ ] Compress (gzip)                     │
│                                          │
│ Base URI: [ http://example.org/onto/ ]   │
│                                          │
│                        [Cancel] [Export] │
└──────────────────────────────────────────┘
```

### Модальное окно импорта

```
┌──────────────────────────────────────────┐
│ Import Ontology                      ✕   │
├──────────────────────────────────────────┤
│ ┌──────────────────────────────────────┐ │
│ │  Drop file here or click to browse    │ │
│ │  .ttl, .rdf, .owl, .nt                │ │
│ └──────────────────────────────────────┘ │
│                                          │
│ Mode:  ○ Merge (add to existing)         │
│        ○ Replace (overwrite ontology)    │
│        ○ Validate only                   │
│                                          │
│ [x] Create commit after import           │
│ [ ] Validate before import (SHACL)       │
│                                          │
│ Commit message:                          │
│ ┌──────────────────────────────────────┐ │
│ │ Import from legacy-ontology-v2.ttl    │ │
│ └──────────────────────────────────────┘ │
│                                          │
│                        [Cancel] [Import] │
└──────────────────────────────────────────┘
```

## Зависимости

- **Rust**: сериализатор/десериализатор (высокая производительность)
- **Rio** или **Turtle** crate для парсинга
- **PostgreSQL**: история импорта, создание коммитов
- **Redis**: отслеживание прогресса для больших импортов

## Валидация

### Валидация SHACL (опционально)

```typescript
// Validate against SHACL shapes before import
POST /api/v1/ontologies/{id}/validate-shapes
Body: {
  content: string,
  shapesGraph: string,       // SHACL shapes
  strictMode: boolean         // Fail on violation
}
Response: {
  valid: boolean,
  conforms: boolean,
  results: {
    severity: 'Violation' | 'Warning' | 'Info',
    path: string,
    focusNode: string,
    value: any,
    message: string
  }[]
}
```

## Целевые показатели производительности

| Операция | Размер | Цель |
|-----------|------|--------|
| Экспорт 100K triples | 100,000 | < 10 sec |
| Экспорт 1M triples | 1,000,000 | < 60 sec (streaming) |
| Импорт 100K triples | 100,000 | < 15 sec |
| Валидация 100K triples | 100,000 | < 5 sec |
