# ADR-DES.INFRA.sustainability-energy-efficiency-strategy

**Дата:** 2026-05-10  
**Статус:** Принято

## Контекст

Экологические требования к программному обеспечению становятся важными для корпоративных клиентов, крупных cloud providers и государственных тендеров в ЕС. Для VEDO Core эти требования пока не блокируют MVP, но early adoption энергоэффективных практик даёт конкурентное преимущество и снижает стоимость эксплуатации.

## Требование-источник
- [sustainability-energy-efficiency.md](requirements/REQ-NFR.INFRA.sustainability-energy-efficiency.md)

## Решение

Принять non-blocking sustainability strategy для VEDO Core с целевыми метриками энергоэффективности и архитектурными мерами по их достижению.

Раннее внедрение энергоэффективных практик даёт конкурентное преимущество для корпоративных клиентов и государственных тендеров ЕС, не блокируя MVP. Целевые метрики (Energy per Request < 0.5 Дж, Carbon per User < 100 г CO2e/мес) задают измеримый ориентир без жёстких gates. Rust для CPU-sensitive сервисов, Go для I/O, HPA и Redis caching снижают потребление ресурсов естественным образом, без отдельных sustainability-инициатив.

Использовать Intel RAPL, PowerAPI или Kepler для измерения energy per request. Для carbon reporting применять облачные инструменты провайдера (AWS Carbon Footprint Tool, Azure Emissions Impact Dashboard, Google Cloud Carbon Sense). Sustainability metrics сделать optional для MVP и включать для корпоративных клиентов по запросу.

## Рассмотренные альтернативы

| Альтернатива | Причина отклонения |
|--------------|--------------------|
| Игнорировать sustainability полностью | Потеря competitive advantage и готовности к корпоративным/ЕС green criteria |
| Сделать carbon targets hard gate для MVP | Преждевременно и может замедлить MVP без требования заказчика |
| Требовать только green cloud providers | Может конфликтовать с 152-ФЗ, GDPR data residency и политикой заказчика |
| Оптимизировать только инфраструктуру, не код | Не покрывает busy loops, real-time analytics и CPU-heavy ontology operations |
| Вести отчётность вручную | Непрактично для корпоративных клиентов; лучше опираться на cloud tools и Grafana dashboards |

## Последствия

**Положительные последствия:**
- VEDO Core получает понятные energy/carbon targets без блокировки MVP.
- Корпоративные заказчики могут получить carbon reporting и dashboards.
- Rust/Go/HPA/caching decisions получают дополнительное sustainability rationale.
- Green region selection становится осознанной опцией развёртывания.

**Отрицательные последствия:**
- Потребуется дополнительный observability слой для energy metrics.
- Cloud carbon tools отличаются по провайдерам и регионам.
- Некоторые targets могут быть трудно измеримы в on-premise и air-gapped environments.

**Меры снижения рисков:**
- Делать sustainability metrics optional для MVP и включать их для корпоративных клиентов по запросу.
- Использовать Kepler или Scaphandre для Kubernetes/on-premise environments.
- Использовать AWS Customer Carbon Footprint Tool, Azure Emissions Impact Dashboard или Google Cloud Carbon Sense для cloud deployments.
- Явно документировать, что green region selection не должен нарушать data residency.

---
