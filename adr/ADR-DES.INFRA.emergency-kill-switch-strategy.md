# ADR-DES.INFRA.emergency-kill-switch-strategy

**Дата:** 2026-05-13  
**Статус:** Принято

## Контекст

При ошибочном массовом import, некорректном merge, компрометации данных или опасной миграции система должна быстро остановить write-поток, сохранив read-доступ для диагностики. Физическое отключение сервиса может ухудшить ситуацию потерей in-flight транзакций и снижением observability.

## Требование-источник
- [critical-alerts.md](requirements/REQ-NFR.OPS.critical-alerts.md)
- [fault-scenarios.md](requirements/REQ-NFR.INFRA.fault-scenarios.md)
- [observability-stack.md](requirements/REQ-NFR.OPS.observability-stack.md)

## Решение

Основной аварийный выключатель — логический read-only mode L1: vedo-cli emergency readonly, API Gateway отклоняет mutating requests при сохранении read-доступа, целевое время активации ≤ 60 секунд через независимую management точку доступа.

Логический read-only mode останавливает write-поток без потери read-доступа к данным для диагностики, в отличие от физического отключения сервиса, которое теряет in-flight транзакции и observability. L2 (vedo-cli emergency kill --service, ≤ 120 с) и L3 (Network ACL/shutdown pod, ≤ 300 с) остаются запасной вариант, если L1 недоступен.

L1 доступен через независимую management точку доступа с priority path; метрика emergency_readonly_activation_seconds; обратный переход — только через vedo-cli emergency clear после устранения причины; WebSocket-уведомления и audit log фиксируют активацию.

## Рассмотренные альтернативы

| Альтернатива | Причина отклонения |
|--------------|--------------------|
| Сразу физически отключать сервис | Потеря read-доступа, in-flight транзакций и диагностической видимости |
| Только circuit breaker | Не даёт оператору явного аварийного control plane |
| Останавливать только UI | API и CLI продолжают mutating operations |

## Последствия

**Положительные последствия:**
- Write-поток останавливается за ≤ 60 секунд без остановки чтения.
- Инженеры сохраняют доступ к данным для диагностики.
- Fault injection tests получают измеримые критерии FT1–FT3.

**Отрицательные последствия:**
- Нужна независимая management точка доступа.
- API Gateway должен уметь глобально отклонять mutating requests.

**Меры снижения рисков:**
- L2/L3 остаются запасной вариант, если L1 недоступен.
- WebSocket-уведомления и audit log фиксируют включение режима.

---
