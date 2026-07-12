<a id="us-abox.individuals.navigate-link"></a>
# US-abox.individuals.navigate-link: Переход к связанному индивиду по ссылочному свойству

```gherkin
@US-abox.individuals.navigate-link @UC-abox.individuals.navigate-linked-individuals @P1 @abox @navigation
Feature: US-abox.individuals.navigate-link Переход к связанному индивиду по ссылочному свойству

  Background:
    Given пользователь аутентифицирован как "Инженер знаний"
    And открыта карточка индивида "ТЭС-Новосибирская-2024"
    And свойство "имеетРегион" ссылается на индивид "Регион-Сибирь"

  Scenario: Пользователь переходит к связанному индивиду без ручного поиска
    When пользователь выбирает ссылку в свойстве "имеетРегион"
    Then система открывает карточку индивида "Регион-Сибирь"
    And в карточке отображаются его свойства и связи
```
