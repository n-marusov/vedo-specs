<a id="us-api.cypher.execute"></a>
# US-api.cypher.execute: Выполнение CYPHER MATCH запроса через API

```gherkin
@US-api.cypher.execute @UC-browse.search.execute-sparql-query-through-gui @P1 @api @cypher @query
Feature: US-api.cypher.execute Выполнение CYPHER MATCH запроса через API

  Background:
    Given разработчик имеет доступ к REST API VEDO
    And онтология "test" содержит классы и индивиды

  Scenario: Разработчик выполняет CYPHER MATCH запрос
    Given API-ключ имеет права "Viewer"
    When разработчик отправляет POST-запрос на "/api/v1/cypher"
    And тело запроса: { "query": "MATCH (n) RETURN n LIMIT 10" }
    Then API возвращает HTTP 200 OK
    And тело ответа содержит "results" с массивом строк
    And ответ содержит execution_time_ms и triple_count

  Scenario: API отклоняет CYPHER запрос на запись
    Given API-ключ имеет права "Viewer"
    When разработчик отправляет POST-запрос на "/api/v1/cypher"
    And тело запроса: { "query": "CREATE (n:Test {id: 'x'})" }
    Then API возвращает HTTP 400 Bad Request
    And тело ответа содержит код "GATEWAY-QUERY-READONLY"

  Scenario: API отклоняет некорректный CYPHER синтаксис
    Given API-ключ имеет права "Viewer"
    When разработчик отправляет POST-запрос на "/api/v1/cypher"
    And тело запроса: { "query": "MATCH INVALID SYNTAX" }
    Then API возвращает HTTP 400 Bad Request
    And тело ответа содержит код ошибки

  Scenario: API возвращает 401 без аутентификации
    Given API-запрос не содержит JWT-токен
    When разработчик отправляет POST-запрос на "/api/v1/cypher"
    Then API возвращает HTTP 401 Unauthorized
```
