<a id="us-io.document.extract-pdf-docx"></a>
# US-io.document.extract-pdf-docx: Извлечение онтологии из PDF и DOCX

```gherkin
@US-io.document.extract-pdf-docx @UC-io.import.extract-ontology-from-document @P1 @io @document @llm
Feature: US-io.document.extract-pdf-docx Извлечение онтологии из PDF и DOCX

  Background:
    Given пользователь аутентифицирован как "Аналитик"
    And открыта онтология "TestOntology"

  Scenario: Успешное извлечение из PDF
    When пользователь загружает файл "technical-spec.pdf"
    Then система извлекает текст с учётом заголовков и параграфов
    And LLM генерирует sequence: классы из заголовков H1-H2, описания из параграфов
    And система возвращает preview с извлечённой онтологией

  Scenario: Успешное извлечение из DOCX
    When пользователь загружает файл "documentation.docx"
    Then система извлекает структуру: заголовки, параграфы, таблицы
    And таблицы в документе преобразуются в datatype properties
    And система возвращает preview

  Scenario: PDF с паролем
    When пользователь загружает защищённый паролем PDF
    Then система отклоняет файл: "PDF защищён паролем. Снимите защиту и повторите."

  Scenario: Сканированный PDF (без OCR)
    When пользователь загружает сканированный PDF без текстового слоя
    Then система сообщает: "PDF не содержит текстового слоя. Используйте OCR перед загрузкой."

  Scenario: Большой документ (150 страниц)
    When пользователь загружает PDF на 150 страниц
    Then система обрабатывает документ с отображением прогресса
    And preview доступен через 40 секунд (p95)
```
