# ADR-DES.INFRA.doc-extractor-service-strategy — Стратегия сервиса извлечения онтологии из документов

**Статус:** ПРЕДЛОЖЕНО
**Дата:** 2026-07-16

## Контекст

VEDO Hub внедряет извлечение OWL-онтологии из загружаемых документов (F14.4, F14.6) — функцию, позволяющую пользователям загружать файлы в любом формате (MD, TXT, PDF, DOCX, JSON, XML, CSV, XLSX) и автоматически получать OWL-онтологию через LLM с предпросмотром и подтверждением.

**Проблемы:**

1. **Многообразие форматов:** Каждый формат требует специализированного парсера (PDF — pypdf, DOCX — python-docx, JSON/XML — рекурсивный обход, XLSX — openpyxl). Разные языки имеют разную зрелость библиотек.
2. **LLM-интеграция:** Вызов LLM-провайдера требует HTTP-клиента, управления контекстом, чанкинга, обработки ошибок и повторных попыток.
3. **Sequence-формат:** LLM должен возвращать не OWL/Turtle (сложный синтаксис), а sequence операций (JSON) — нужна валидация schema.
4. **Предпросмотр:** Пользователь должен видеть результат до записи в Neo4j — нужен двухфазный протокол (preview → confirm).
5. **Атомарность:** Запись в Neo4j должна быть транзакционной — нужен gRPC-контракт с ontology-service.

**Целевые принципы:**

1. **Правильный язык для задачи:** Парсинг документов + LLM → Python (богатая экосистема).
2. **Следование архитектуре:** Никакого прямого доступа к Neo4j, только через ontology-service (gRPC ApplySequence).
3. **Прозрачность:** Предпросмотр, индикатор прогресса, отчёт об ошибках.
4. **Контроль:** Пользователь редактирует sequence перед импортом.
5. **Безопасность:** Сканирование на вирусы, защита от XXE, отключение макросов.

## Требование-источник

- `REQ-FUN.API.doc-extract-supported-formats`
- `REQ-FUN.API.doc-extract-flow`
- `REQ-FUN.API.doc-extract-sequence-schema`
- `REQ-NFR.API.doc-extract-performance`
- `REQ-NFR.SECURITY.doc-extract-security`
- `REQ-USR.UI.doc-extract-upload`
- `specs/vision.md` (F14.4, F14.6, F14.7, F14.8)

## Решение

Внедрить **document-extractor service** — новый Python-микросервис для извлечения OWL-онтологии из документов, общающийся с ontology-service через gRPC.

### 1. Компоненты архитектуры

```
┌──────────────┐     HTTP/REST (через API Gateway)
│   Frontend   │──────────────────────────────────┐
│   (Vue 3)    │                                  │
└──────────────┘                                  ▼
                                            ┌──────────┐
                                            │  API     │
                                            │  Gateway │
                                            │  (Go)    │
                                            └────┬─────┘
                                                 │ HTTP
                                                 ▼
┌───────────────────────────────────────────────────────────┐
│                document-extractor service                  │
│                     (Python)                               │
│                                                           │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐ │
│  │Extractors│  │  LLM     │  │Sequence  │  │ gRPC     │ │
│  │• MD/TXT  │─▶│ Client   │─▶│Validator │─▶│ Client   │ │
│  │• PDF     │  │• OpenAI  │  │• JSON    │  │(to onto- │ │
│  │• DOCX    │  │• Claude  │  │  Schema  │  │ logy     │ │
│  │• JSON/XML│  │• Litellm │  │• Ref.    │  │ service) │ │
│  │• XLSX/CSV│  └──────────┘  │  integr. │  └──────────┘ │
│  └──────────┘                └──────────┘               │
└────────────────────────┬──────────────────────────────────┘
                         │ gRPC ApplySequence
                         ▼
┌───────────────────────────────────────────────────────────┐
│                 ontology-service (Rust)                    │
│                                                           │
│  gRPC handler: ApplySequence(Sequence) → ApplyReport      │
│  • Validate each step                                      │
│  • BEGIN Neo4j TRANSACTION                                 │
│  • foreach step: execute Cypher                            │
│  • COMMIT                                                   │
│  • gRPC → versioning-service: create commit               │
└───────────────────────────────────────────────────────────┘
```

### 2. Поток извлечения

```
1. Frontend → API Gateway: POST /api/v1/ontologies/{id}/extract-from-document
   (multipart: file + strategy + optional system_prompt)

2. API Gateway → document-extractor: HTTP прокси

3. document-extractor:
   a. Extractors: определение формата → извлечение текста/структуры
   b. LLM Client: отправка текста LLM → получение sequence (JSON)
   c. Sequence Validator: проверка schema, ссылочной целостности
   d. Response: preview sequence (без записи в Neo4j)

4. Frontend: отображение preview с возможностью редактирования

5. Frontend → API Gateway: POST /api/v1/ontologies/{id}/confirm-extraction
   { document_id, edited_sequence, commit_message }

6. API Gateway → document-extractor: HTTP прокси

7. document-extractor → ontology-service:
   gRPC: ApplySequence(ontology_id, steps, commit_message)

8. ontology-service:
   a. BEGIN Neo4j TRANSACTION
   b. foreach step: validate preconditions → execute Cypher
   c. COMMIT
   d. gRPC → versioning-service: create commit
   e. Response: ApplyReport { commit_id, applied, skipped, errors }
```

### 3. Sequence JSON Schema

LLM возвращает JSON строгой структуры (см. `REQ-FUN.API.doc-extract-sequence-schema`). Ключевые типы операций:

| op | Описание | Обязательные поля |
|---|---|---|
| `create_class` | Создать класс | id, [parent] |
| `set_parent` | Установить родителя | id, parent |
| `create_object_property` | Создать связь | id, domain, range |
| `create_datatype_property` | Создать атрибут | id, domain, data_type |
| `add_annotation` | Добавить аннотацию | id, range, value |
| `create_individual` | Создать индивида | id, parent |

### 4. Форматы и парсеры

| Формат | Парсер | Примечание |
|--------|--------|------------|
| `.md` | `markdown` (Python) | Заголовки → классы |
| `.txt` | Plain text | Сплошной текст → LLM |
| `.pdf` | `pypdf` / `pdfplumber` | Только текстовые PDF |
| `.docx` | `python-docx` | Заголовки, таблицы |
| `.json` | `json` (stdlib) | Ключи → классы |
| `.xml` | `xml.etree` | Теги → классы, атрибуты → свойства |
| `.csv` | `csv` (stdlib) | LLM + маппинг колонок |
| `.xlsx` | `openpyxl` | LLM + маппинг колонок |

### 5. LLM-интеграция

- Используется существующий LLM Policy Router (см. `REQ-FUN.API.llm-policy`).
- Промпт: system prompt (изолированный, запрещающий вредоносный вывод) + содержимое документа.
- Чанкинг: при превышении контекста LLM документ разбивается на секции с сохранением структуры.
- Retry: до 3 попыток при ошибке LLM.

### 6. Обработка ошибок

| Ошибка | HTTP Status | Действие |
|--------|------------|----------|
| Неподдерживаемый формат | 400 | Список поддерживаемых форматов |
| Файл > 20 MB | 413 | Предложить асинхронный режим |
| PDF с паролем | 400 | «Снимите защиту» |
| Пустой файл | 400 | «Файл пуст» |
| LLM timeout / error | 502 / 504 | Retry до 3 раз, затем ошибка |
| Невалидный sequence | 422 | «Не удалось извлечь онтологию» |
| gRPC ApplySequence error | 502 | Детали ошибки ontology-service |

### 7. Безопасность

- Virus scan перед обработкой.
- PDF: блокировка JavaScript, external refs.
- DOCX: отключение макросов.
- XML: защита от XXE.
- Временные файлы: 24h TTL, автоматическое удаление.
- Аудит: все загрузки логируются.

## Рассмотренные альтернативы

1. **Расширение ontology-service (Rust) для парсинга документов:**
   - *Против:* Rust не имеет зрелых библиотек для PDF/DOCX, разработка парсеров займёт значительно больше времени. LLM-интеграция в Rust — меньше экосистемы.

2. **Расширение ticket-classifier (Python) для document extraction:**
   - *Против:* ticket-classifier — узкоспециализированный сервис для классификации тикетов. Смешение ответственности нарушает принцип единой обязанности.

3. **Обработка на фронтенде (Vue 3):**
   - *Против:* LLM API keys не должны быть на клиенте. Парсинг PDF/DOCX в браузере ограничен. Невозможно обеспечить безопасность.

4. **LLM → Turtle → POST /import (существующий endpoint):**
   - *Против:* LLM с трудом генерирует синтаксически корректный Turtle. Невозможен предпросмотр. All-or-nothing импорт без контроля каждого шага.

## Последствия

**Положительные:**
- ✅ Правильный язык (Python) для задачи парсинга + LLM.
- ✅ Чёткое разделение ответственности: extractor = NLP, ontology-service = граф.
- ✅ Двухфазный протокол (preview → confirm) даёт пользователю контроль.
- ✅ Sequence JSON проще для LLM, чем Turtle.
- ✅ Атомарное применение через gRPC в одной Neo4j транзакции.
- ✅ Следование архитектурному принципу «Service owns its data».

**Отрицательные:**
- ❌ Новый сервис = новая точка отказа, развёртывания, мониторинга.
- ❌ Дополнительная сетевая задержка (frontend → gateway → extractor → LLM → extractor → gRPC → ontology-service → Neo4j).
- ❌ Python-сервис требует управления зависимостями (poetry/pip), что добавляет overhead.
- ❌ Необходимость синхронизации protobuf-контрактов между Python (extractor) и Rust (ontology-service).

**Нейтральные:**
- ◎ Временное хранение файлов (24h) требует object storage или локальной FS.
- ◎ Необходимость мониторинга нового сервиса (health, ready, metrics как у всех сервисов VEDO).

## Связанные ADR

- `ADR-DES.API.llm-policy-router-strategy` — LLM Policy Router
- `ADR-DES.INFRA.monolith-vs-microservices` — Микросервисная архитектура
- `ADR-IMPL.STACK.microservice-language-stack-strategy` — Выбор языка для сервисов
- `ADR-IMPL.STACK.ontology-rust-strategy` — Почему ontology-service на Rust
- `ADR-DES.UI.excel-import-strategy` — Стратегия импорта Excel (дополняется данным ADR)

## Чек-лист реализации

- [ ] Создать Python-сервис `src/services/document-extractor/` с структурой:
  - `main.py` — HTTP-сервер (FastAPI / uvicorn)
  - `extractors/` — парсеры форматов (base.py, text.py, markdown.py, pdf.py, docx.py, structured.py)
  - `llm/` — LLM-клиент (client.py, prompts.py)
  - `ontology/` — валидатор sequence (builder.py, validator.py)
  - `handlers/` — HTTP-handlers (extract.py, confirm.py)
  - `proto/` — gRPC-клиент к ontology-service
  - `Dockerfile`, `pyproject.toml`, `requirements.txt`
- [ ] Определить protobuf-контракт `ApplySequence` (gRPC)
- [ ] Реализовать gRPC-handler в ontology-service (Rust, tonic)
- [ ] Добавить роут в API Gateway для прокси document-extractor
- [ ] Реализовать UI-компоненты (drag-n-drop, прогресс, preview)
- [ ] Интегрировать virus scan (ClamAV)
- [ ] Написать тесты (unit + integration)
- [ ] Обновить Docker Compose
