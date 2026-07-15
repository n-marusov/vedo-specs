<a id="us-io.document.extract-md-txt"></a>
# US-io.document.extract-md-txt: Извлечение онтологии из текстовых файлов

```gherkin
@US-io.document.extract-md-txt @UC-io.import.extract-ontology-from-document @P1 @io @document @llm
Feature: US-io.document.extract-md-txt Извлечение онтологии из MD и TXT

  Background:
    Given пользователь аутентифицирован как "Аналитик"
    And открыта онтология "TestOntology"

  Scenario: Успешное извлечение из Markdown
    When пользователь загружает файл "specification.md" (Markdown)
    Then система возвращает preview: 8 классов, 5 свойств, 3 уровня иерархии
    And каждый класс имеет label, извлечённый из заголовка Markdown
    And каждый пункт списка под заголовком добавлен как свойство класса

  Scenario: Успешное извлечение из Plain Text
    When пользователь загружает файл "requirements.txt"
    Then система возвращает preview с извлечёнными классами и свойствами
    And пользователь может отредактировать label перед подтверждением

  Scenario: Ошибка при пустом файле
    When пользователь загружает пустой файл "empty.txt"
    Then система сообщает: "Файл не содержит текста для извлечения"

  Scenario: Ошибка при превышении размера
    When пользователь загружает файл размером 25 MB
    Then система сообщает: "Файл превышает 20 MB. Используйте асинхронную загрузку."
    And предлагает переключиться в асинхронный режим
```
