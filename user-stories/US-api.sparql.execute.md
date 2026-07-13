<a id="us-api.sparql.execute"></a>
# US-api.sparql.execute: Выполнение SPARQL SELECT запроса через API

```gherkin
@US-api.sparql.execute @UC-browse.search.execute-sparql-query-through-gui @P1 @api @sparql @query
Feature: US-api.sparql.execute Выполнение SPARQL SELECT запроса через API

  Background:
    Given разработчик имеет доступ к REST API VEDO
    And онтология "test" содержит классы и свойства

  Scenario: Разработчик выполняет SPARQL SELECT запрос
    Given API-ключ имеет права "Viewer"
    When разработчик отправляет POST-запрос на "/api/v1/sparql"
    And тело запроса: { "query": "SELECT ?s ?p ?o WHERE { ?s ?p ?o } LIMIT 10" }
    Then API возвращает HTTP 200 OK
    And тело ответа содержит "results" с массивом строк
    And ответ содержит execution_time_ms и triple_count

  Scenario: API отклоняет SPARQL INSERT запрос
    Given API-ключ имеет права "Viewer"
    When разработчик отправляет POST-запрос на "/api/v1/sparql"
    And тело запроса: { "query": "INSERT DATA { <urn:a> <urn:b> <urn:c> }" }
    Then API возвращает HTTP 400 Bad Request
    And тело ответа содержит код "GATEWAY-QUERY-READONLY"

  Scenario: API отклоняет некорректный SPARQL синтаксис
    Given API-ключ имеет права "Viewer"
    When разработчик отправляет POST-запрос на "/api/v1/sparql"
    And тело запроса: { "query": "SELECT INVALID SYNTAX" }
    Then API возвращает HTTP 400 Bad Request
    And тело ответа содержит код ошибки

  Scenario: API возвращает 401 без аутентификации
    Given API-запрос не содержит JWT-токен
    When разработчик отправляет POST-запрос на "/api/v1/sparql"
    Then API возвращает HTTP 401 Unauthorized
```
