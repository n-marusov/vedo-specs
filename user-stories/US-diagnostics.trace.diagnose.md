<a id="us-diagnostics.trace.diagnose"></a>
# US-diagnostics.trace.diagnose: Диагностика инцидента по trace_id

```gherkin
@US-diagnostics.trace.diagnose @UC-diagnostics.trace.diagnose-incident-by-trace-id @P0 @support @diagnostics @otel @vedo-cli
Feature: US-diagnostics.trace.diagnose Диагностика инцидента по trace_id

  Background:
    Given инженер поддержки имеет доступ к observability stack
    And Grafana Tempo, Loki и Prometheus доступны

  Scenario: Инженер поддержки локализует ошибку по trace_id
    Given инженер поддержки получил trace_id "abc123" из P0-алерта
    When инженер поддержки выполняет команду "vedo-cli diagnose trace --id abc123"
    Then инструмент загружает trace из Tempo
    And показывает span с ошибкой
    And загружает связанные логи из Loki
    And отображает метрики CPU, memory, latency и error rate за время инцидента
    And инженер поддержки получает данные для сокращения MTTD
```
