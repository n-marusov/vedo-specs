# Поток извлечения онтологии из документа

| Атрибут | Значение |
|---------|----------|
| **ID** | REQ-FUN.API.doc-extract-flow |
| **Уровень** | FUN |
| **Атрибут качества** | Functionality |
| **Приоритет** | P0 |
| **Статус** | ЧЕРНОВИК |
| **Источник** | vision.md (F14.6) |

---

## Назначение

Определить полный поток извлечения OWL-онтологии из загружаемого документа.

## Требование

Система реализует следующий поток для извлечения онтологии из документа:

### Flow: document-extractor

```
1. POST /api/v1/ontologies/{ontology_id}/extract-from-document
   (multipart: file + options)
       │
       ▼
2. document-extractor:
   a. Определение MIME-типа и формата файла
   b. Извлечение текста с учётом форматирования (заголовки, параграфы, таблицы)
   c. Чанкинг текста при превышении контекста LLM
   d. Отправка chunk(ов) LLM-провайдеру с промптом на извлечение онтологии
   e. LLM возвращает sequence операций (JSON)
   f. Валидация sequence (JSON Schema + ссылочная целостность)
       │
       ▼
3. Response: { status: "preview", document_id, sequence: [...] }
   (фронтенд показывает preview)
       │
       ▼
4. POST /api/v1/ontologies/{ontology_id}/confirm-extraction
   { document_id, edited_sequence, commit_message }
       │
       ▼
5. document-extractor → ontology-service:
   gRPC ApplySequence{ ontology_id, steps, commit_message }
       │
       ▼
6. ontology-service:
   a. BEGIN Neo4j TRANSACTION
   b. foreach step: validate → execute Cypher
   c. COMMIT
   d. gRPC: create commit in versioning-service
       │
       ▼
7. Response: { commit_id, total_steps, applied, skipped, errors }
```

### Tребования к шагам

**Шаг 1 (загрузка):**
- Поддержка multipart/form-data для загрузки файла
- Параметры: strategy (merge/replace/version), system_prompt (optional)

**Шаг 2b (извлечение текста):**
- MD: парсинг заголовков (#, ##) → классы, списки (-, *) → свойства
- DOCX: чтение стилей заголовков, параграфов, таблиц
- PDF: извлечение текста с позиционированием, группировка по заголовкам
- JSON/XML: рекурсивный обход структуры

**Шаг 2d (LLM):**
- Sequence-запрос должен быть в строгом JSON-формате (см. REQ-FUN.API.doc-extract-sequence-schema)
- LLM-провайдер конфигурируется через общий LLM Policy (см. REQ-FUN.API.llm-policy)

**Шаг 2f (валидация):**
- Проверка JSON Schema
- Проверка ссылочной целостности: все domain/range/id ссылаются на существующие классы
- Проверка на циклические иерархии (класс не может быть родителем самого себя)

**Шаг 4 (подтверждение):**
- Пользователь может изменить sequence перед подтверждением
- Пользователь может удалить отдельные шаги
- Пользователь может задать commit_message

**Шаг 5-6 (применение):**
- Все steps применяются в одной Neo4j транзакции
- При ошибке валидации любого шага — полный rollback транзакции
- При успехе — создаётся коммит в versioning-service

## Критерии приёмки

1. Пользователь загружает файл → получает preview sequence.
2. В preview пользователь видит классы, свойства, связи без знания OWL.
3. Пользователь может удалить/изменить шаги в preview.
4. После подтверждения sequence атомарно применяется к онтологии.
5. Создаётся коммит с сообщением «Извлечено из <filename>».
6. Исходный файл прикрепляется к коммиту как артефакт.
