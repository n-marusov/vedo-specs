<a id="us-api.ontologies.read-rest"></a>
# US-api.ontologies.read-rest: Чтение списка онтологий, классов, свойств и индивидов через REST API

```gherkin
@US-api.ontologies.read-rest @UC-api.integration.integrate-through-platform-apis @P1 @api @rest @read
Feature: US-api.ontologies.read-rest Чтение списка онтологий, классов, свойств и индивидов через REST API

  Background:
    Given разработчик имеет доступ к REST API VEDO
    And существует онтология "test" с классами, свойствами и индивидами

  Scenario: Разработчик получает список онтологий
    Given API-ключ имеет права "Viewer"
    When разработчик отправляет GET-запрос на "/api/v1/ontologies"
    Then API возвращает HTTP 200 OK
    And тело ответа содержит массив онтологий
    And каждый элемент содержит id, title и counts

  Scenario: Разработчик получает классы онтологии
    Given онтология "test" содержит классы "Person" и "Organization"
    When разработчик отправляет GET-запрос на "/api/v1/ontologies/test/classes"
    Then API возвращает HTTP 200 OK
    And тело ответа содержит классы "Person" и "Organization"
    And ответ содержит пагинацию

  Scenario: Разработчик получает детали класса с иерархией
    Given существует класс "Student" с родителем "Person"
    When разработчик отправляет GET-запрос на "/api/v1/ontologies/test/classes/Student"
    Then API возвращает HTTP 200 OK
    And тело ответа содержит id, label, ancestors и descendants

  Scenario: Разработчик получает индивидов класса
    Given существует класс "Person" с 5 индивидами
    When разработчик отправляет GET-запрос на "/api/v1/ontologies/test/individuals?class_id=Person"
    Then API возвращает HTTP 200 OK
    And тело ответа содержит массив индивидов
    And ответ содержит total = 5

  Scenario: REST API возвращает 401 без аутентификации
    Given API-запрос не содержит JWT-токен
    When разработчик отправляет GET-запрос на "/api/v1/ontologies"
    Then API возвращает HTTP 401 Unauthorized
```
