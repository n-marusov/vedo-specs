# ADR-DES.INFRA.telemetry-retention-strategy

**Дата:** 2026-05-10  
**Статус:** Принято

## Контекст

VEDO Core собирает разные типы telemetry: audit logs, application logs, HTTP/API Gateway access logs, distributed traces и metrics. Единый retention для всех типов телеметрии не подходит: audit logs имеют compliance-ценность и должны храниться дольше, traces имеют огромный объём и быстро теряют ценность, metrics нужны для capacity planning и трендов.

## Требование-источник
- [log-retention.md](requirements/REQ-NFR.OPS.log-retention.md)

## Решение

Принять многоуровневую retention policy для observability data с раздельным сроком хранения для audit logs, application logs, access logs, traces и metrics.

Раздельные retention windows соответствуют разной ценности и стоимости каждого типа телеметрии: audit logs получают compliance-ready lifecycle до 5 лет, а traces и access logs не раздувают storage cost. Metrics сохраняют годовые тренды для capacity planning без хранения raw samples целый год. On-premise заказчики могут адаптировать retention под свои политики через конфигурацию без изменения кода.

Маршрутизировать audit logs в отдельный Loki tenant с увеличенным retention. Application logs, access logs и traces использовать с отдельными retention windows. Prometheus raw metrics хранить 30 дней с remote write в Thanos или Victoria Metrics для долгосрочных aggregated metrics.

## Рассмотренные альтернативы

| Альтернатива | Причина отклонения |
|--------------|--------------------|
| Единый retention 30 дней для всех логов и трейсов | Недостаточно для audit/compliance и избыточно для traces |
| Хранить все telemetry 1 год active | Слишком дорого из-за traces и HTTP access logs |
| Хранить все telemetry 7 дней | Недостаточно для расследования инцидентов, audit и корпоративных compliance |
| Не архивировать audit logs | Неприемлемо для 152-ФЗ, корпоративных политик, ISO 27001 и SOC 2 сценариев |
| Не делать переопределения заказчика | Непрактично для on-premise и регулируемых корпоративных развёртываний |

## Последствия

**Положительные последствия:**
- Retention соответствует разной ценности и стоимости разных типов telemetry.
- Audit logs получают отдельный compliance-ready lifecycle.
- Traces и access logs не раздувают storage cost сверх необходимости.
- Metrics сохраняют годовые тренды без хранения raw samples целый год.
- On-premise заказчики могут адаптировать retention под свои политики.

**Отрицательные последствия:**
- Конфигурация observability становится сложнее: разные tenants, retention windows и archive tiers.
- Нужен capacity planning для переопределений заказчика и корпоративного хранения.
- Для audit cold archive требуется отдельная процедура выгрузки, хранения и проверки доступности.

**Меры снижения рисков:**
- Поставлять default Loki, Prometheus и Tempo retention configuration.
- Разделять audit logs и application logs через OpenTelemetry Collector routing по `log.type`.
- Для long-term metrics использовать remote write и aggregated storage.
- Документировать влияние переопределений заказчика на disk capacity и стоимость.
- Для 152-ФЗ предусмотреть отдельный cold archive, например S3 Glacier или управляемый заказчиком эквивалент.

---
