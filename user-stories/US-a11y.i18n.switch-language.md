<a id="us-a11y.i18n.switch-language"></a>
# US-a11y.i18n.switch-language: Переключение языка интерфейса с сохранением контекста

```gherkin
@US-a11y.i18n.switch-language @A9 @P1 @accessibility @i18n
Feature: US-a11y.i18n.switch-language Переключение языка интерфейса с сохранением контекста

  Background:
    Given пользователь использует keyboard-only навигацию
    And интерфейс поддерживает русский и английский языки

  Scenario: Пользователь переключает язык с клавиатуры
    When пользователь фокусируется на переключателе языка через Tab
    And выбирает другой язык
    Then интерфейс переключается на выбранный язык
    And текущий контекст навигации сохраняется
    And screen reader объявляет смену языка
```
