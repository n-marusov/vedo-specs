# ADR-DES.INFRA.vedo-cli-diagnostics-entrypoint

**Дата:** 2026-05-12  
**Статус:** Принято

## Контекст

Инженер поддержки не может быстро локализовать проблему из-за разрозненности данных: логи находятся в Loki, трейсы в Tempo, метрики в Prometheus, а ручная диагностика требует переключения между Grafana dashboards и знания структуры сервисов VEDO Core.

## Требование-источник
- [critical-alerts.md](requirements/REQ-NFR.OPS.critical-alerts.md)
- [observability-stack.md](requirements/REQ-NFR.OPS.observability-stack.md)

## Решение как единую административную точку входа в OpenTelemetry stack для корреляции traces, логов и метрик по `trace_id`.

Единая точка входа снижает время диагностики инцидентов (MTTD P0 ≤ 10 минут) и исключает переключение между Tempo, Loki и Prometheus. Команда становится воспроизводимой и аудируемой — её можно включать в runbooks, CI smoke checks и скрипты поддержки. Опциональная интеграция LLM даёт рекомендации по устранению, но не является обязательной, что сохраняет работоспособность в air-gapped средах.

Реализовать `vedo-cli diagnose trace --id <trace_id>`, которая запрашивает trace из Tempo, извлекает ошибочный `span_id` и связанные span attributes, затем запрашивает логи из Loki и метрики из Prometheus за соответствующий интервал. LLM-режим сделать опциональным с явным флагом и отключать в air-gapped среде.

## Рассмотренные альтернативы

| Альтернатива | Причина отклонения |
|--------------|--------------------|
| Использовать Grafana вручную | Требует переключения между интерфейсами, плохо автоматизируется и не гарантирует MTTD P0 ≤ 10 минут |
| Писать отдельные scripts для Tempo/Loki/Prometheus | Дублирует логику, ухудшает аудит и усложняет поддержку разных моделей развёртывания |
| Делать диагностику только в web UI | Не подходит для blind environments, SSH-only операций, runbooks и air-gapped администрирования |
| Требовать LLM всегда | Неприемлемо для air-gapped и регулируемых развёртываний, где внешний LLM может быть запрещён |

## Последствия

**Положительные последствия:**
- Инженер поддержки получает одну команду для корреляции traces/logs/metrics.
- Time-to-Diagnose становится проверяемым эксплуатационным NFR.
- Диагностика может использоваться в runbooks, CI smoke checks и customer support scripts.
- Air-gapped deployments сохраняют базовую диагностику без внешних LLM вызовов.

**Отрицательные последствия:**
- Все сервисы обязаны корректно инструментировать traces, logs и metrics с correlation IDs.
- `vedo-cli` должен знать схемы labels/attributes в Tempo, Loki и Prometheus.
- Рекомендации LLM требуют отдельной политики безопасности, redaction и управления секретами.

**Меры снижения рисков:**
- Закрепить обязательные span attributes, log labels и metric labels в observability contracts.
- Маскировать PII/secrets перед передачей диагностического контекста в LLM.
- Включить `vedo-cli diagnose` сценарии в validation gates для observability operations.

---
