<a id="us-diagnostics.trace.llm-recommend"></a>
# US-diagnostics.trace.llm-recommend: LLM-рекомендации по устранению инцидента

```gherkin
@US-diagnostics.trace.llm-recommend @UC-diagnostics.trace.diagnose-incident-by-trace-id @P1 @support @diagnostics @llm @vedo-cli
Feature: US-diagnostics.trace.llm-recommend LLM-рекомендации по устранению инцидента

  Background:
    Given инженер поддержки имеет доступ к observability stack
    And LLM diagnostics включён политикой окружения

  Scenario: Инженер поддержки получает рекомендации по инциденту
    Given инженер поддержки получил trace_id "abc123" из алерта
    When инженер поддержки выполняет команду "vedo-cli diagnose trace --id abc123 --recommend"
    Then vedo-cli маскирует секреты и PII в диагностическом контексте
    And вызывает LLM для анализа ошибки
    And показывает конкретные рекомендации и команды устранения
```
