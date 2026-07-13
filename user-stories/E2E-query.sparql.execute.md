<a id="e2e-query.sparql.execute"></a>
# E2E-query.sparql.execute: Выполнение SPARQL и CYPHER запросов

```gherkin
@E2E-query.sparql.execute @e2e @query @sparql @cypher @P1
Feature: E2E-query.sparql.execute Выполнение SPARQL и CYPHER запросов

  Scenario: Разработчик выполняет SPARQL SELECT и CYPHER MATCH запросы
    Given разработчик имеет доступ к REST API VEDO
    And онтология "test" содержит классы "Person" и "Organization"
    When разработчик отправляет SPARQL SELECT запрос
    Then ответ содержит результаты в табличном формате
    When разработчик отправляет CYPHER MATCH запрос
    Then ответ содержит результаты в едином формате
    And каждый результат содержит execution_time_ms и triple_count

  Scenario: API отклоняет мутирующий SPARQL запрос
    Given разработчик имеет доступ к REST API VEDO
    When разработчик отправляет SPARQL INSERT запрос
    Then API возвращает HTTP 400
    And ответ содержит код ошибки "GATEWAY-QUERY-READONLY"

  Scenario: API отклоняет запрос с некорректным синтаксисом
    Given разработчик имеет доступ к REST API VEDO
    When разработчик отправляет SPARQL запрос с неверным синтаксисом
    Then API возвращает HTTP 400
    And ответ содержит код ошибки

  Scenario: API возвращает 401 без аутентификации
    Given API-запрос не содержит JWT-токен
    When разработчик отправляет SPARQL запрос
    Then API возвращает HTTP 401 Unauthorized
```
