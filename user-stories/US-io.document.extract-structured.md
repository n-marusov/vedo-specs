<a id="us-io.document.extract-structured"></a>
# US-io.document.extract-structured: Извлечение онтологии из структурированных данных

```gherkin
@US-io.document.extract-structured @UC-io.import.import-structured-data @P1 @io @document @xlsx @csv @json @xml
Feature: US-io.document.extract-structured Извлечение онтологии из JSON, XML, CSV, XLSX

  Background:
    Given пользователь аутентифицирован как "Аналитик"
    And открыта онтология "TestOntology"

  Scenario: Импорт из XLSX с колонками класс/родитель/свойство/тип
    When пользователь загружает XLSX-файл с колонками "Класс, Родитель, Свойство, Тип"
    Then система определяет маппинг колонок автоматически
    And LLM генерирует классы и свойства на основе таблицы
    And preview отображает иерархию классов

  Scenario: Импорт из JSON с вложенной структурой
    When пользователь загружает JSON-файл с вложенными объектами
    Then система преобразует вложенность ключей в иерархию классов
    And значения-примитивы становятся datatype properties
    And ссылки между объектами становятся object properties

  Scenario: Импорт из XML
    When пользователь загружает XML-файл
    Then теги XML становятся классами
    And атрибуты тегов становятся свойствами
    And вложенность тегов определяет иерархию классов

  Scenario: Импорт из CSV
    When пользователь загружает CSV-файл
    Then система автоопределяет разделитель (`,` или `;`)
    And первая строка интерпретируется как заголовки колонок
    And LLM определяет маппинг колонок на классы/свойства
```
