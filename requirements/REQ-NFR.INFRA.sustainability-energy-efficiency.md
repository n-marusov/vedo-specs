# Экологичность и энергоэффективность

| Атрибут | Значение |
|---------|----------|
| **ID** | REQ-NFR.INFRA.sustainability-energy-efficiency |
| **Уровень** | NFR |
| **Атрибут качества** | Physical |
| **Приоритет** | P2 |
| **Статус** | УТВЕРЖДЕНО |
| **Источник** | — |
| **Критерии приёмки** | — |

---

## Назначение

Фиксирует ответ на вопрос S1.3: есть ли требования к экологии, энергоэффективности и carbon footprint для VEDO Core.

Экологические требования к программному обеспечению не являются блокирующими для VEDO Core на текущей стадии, но их учет дает конкурентное преимущество для enterprise, крупных корпораций и государственных тендеров в ЕС, где с 2024 года появляются зеленые критерии.

## Целевые метрики

| Метрика | Определение | Целевое значение для VEDO Core | Метод измерения |
|---------|-------------|-------------------------------|-----------------|
| Energy per Request | Средняя энергия на один API-запрос | < 0.5 Дж/запрос | Intel RAPL, PowerAPI |
| Carbon per User per Month | CO2e на одного активного пользователя в месяц | < 100 г CO2e | Cloud provider carbon tools |
| Energy Proportionality | Энергопотребление при простое / при нагрузке | < 30% от пикового | Kubernetes, kube-state-metrics |
| Compute Utilization | Средняя загрузка CPU в кластере | > 40% | Prometheus, Grafana |

## Архитектурные решения, поддерживающие экологичность

| Решение | Влияние на энергоэффективность | Сложность реализации | Статус в VEDO Core |
|---------|-------------------------------|----------------------|--------------------|
| Rust для ядра: Ontology Service | Высокое: меньше CPU, меньше энергии | Средняя | Реализовано |
| Горизонтальное масштабирование HPA | Среднее: ресурсы потребляются по требованию | Средняя | Реализовано |
| Снижение idle resource usage | Высокое | Низкая | Настроить HPA с нулевыми репликами при отсутствии трафика |
| Go для сетевых сервисов | Среднее | Низкая | Go используется для gateway/network services |
| Архитектура, оптимизированная под concurrency | Среднее | Средняя | RabbitMQ, Kafka |
| Cloud provider с низким PUE | Высокое | Низкая | Выбирать регион с renewable energy |
| Redis caching | Среднее: меньше запросов к БД | Средняя | Реализовано |
| Batch processing вместо real-time для analytics | Высокое | Средняя | Не реализовано, кандидат P3 |

## Энергетические метрики Kubernetes

Инструменты для измерения carbon footprint и энергопотребления:

- Kepler: Kubernetes-based Efficient Power Level Exporter.
- Scaphandre: agent для измерения энергопотребления контейнеров.
- AWS Customer Carbon Footprint Tool для cloud deployments.
- Azure Emissions Impact Dashboard для Azure deployments.
- Google Cloud Carbon Sense для Google Cloud deployments.

Пример Prometheus query для Kepler:

```prometheus
avg(kepler_node_package_joules_total{job="kepler"}) by (pod)
```

Пороговые значения:

| Статус | Energy per Node | Действие |
|--------|-----------------|----------|
| Green | < 500 Wh/day | OK |
| Yellow | 500-1000 Wh/day | Оптимизация |
| Red | > 1000 Wh/day | Срочная оптимизация |

## Carbon Profile Cloud Providers

| Провайдер | Регион | PUE | Возобновляемая энергия | Инструмент |
|-----------|--------|-----|------------------------|------------|
| AWS eu-central-1 | EU Frankfurt | 1.2 | 50% | Customer Carbon Footprint Tool |
| AWS us-east-1 | USA | 1.15 | 30% | Customer Carbon Footprint Tool |
| Yandex Cloud RU | Russia | 1.35 | 0% | Нет отдельного инструмента |
| Azure EU | EU | 1.1 | 60% | Emissions Impact Dashboard |
| Google Cloud EU | EU | 1.09 | 90% | Carbon Sense |

Рекомендация для зеленых развертываний: использовать Google Cloud в Европе, если нет ограничений data residency или customer policy, потому что PUE около 1.09 и высокая доля renewable energy.

## Политика энергоэффективности на уровне кода

Код VEDO Core должен избегать busy loops, постоянной фоновой нагрузки без необходимости и real-time processing там, где достаточно batch processing.

Пример подхода для CPU-intensive операций:

```rust
use tokio::time::Duration;

async fn process_ontology(&self) {
    for node in self.nodes {
        self.expensive_operation(node).await;
        tokio::time::sleep(Duration::from_millis(10)).await;
    }
}
```

Для analytics предпочтителен batch mode, если нет требований real-time:

```yaml
analytics:
  mode: batch
  schedule: "0 * * * *"
```

## Carbon Reporting Для Заказчиков

Для enterprise-клиентов VEDO Core может предоставлять monthly carbon footprint report.

## Область hardware lifecycle

Hardware recycling, e-waste и vendor retirement находятся вне sustainability scope MVP.

- SaaS MVP опирается на public cloud providers; lifecycle физического оборудования и e-waste handling являются ответственностью provider.
- Customer-managed on-premise deployments используют инфраструктуру заказчика; hardware recycling и media disposal являются ответственностью заказчика.
- Managed private cloud hardware retirement, disposal certificates, licensed e-waste operators и disk destruction protocols откладываются до Enterprise/P3 contracts.
- Vendor retirement для VEDO Core не требует MVP process, потому что ПО распространяется под MIT license и может продолжать работать или быть forked.

Стандарты и методологии:

- SCI: Software Carbon Intensity от Green Software Foundation.
- GSF Standard: методология измерения углеродной интенсивности ПО.
- ISO 14064: корпоративная отчетность.

Пример отчета:

```yaml
period: 2025-01
active_users: 2500
cluster_energy: 850 kWh
renewable_energy: 45%
carbon_footprint_total: 340 kg CO2e
carbon_footprint_per_user: 136 g CO2e
measurement_method: AWS Customer Carbon Footprint Tool
pue: 1.18
```

## Формулировка для заказчика

VEDO Core проектируется с учетом принципов энергоэффективности:

- Rust используется для ядра, чтобы снижать CPU usage и энергопотребление.
- HPA позволяет потреблять ресурсы по требованию.
- Возможны зеленые cloud regions, например Google Cloud EU и Azure EU.
- Энергопотребление можно мониторить через Kepler и Grafana dashboards.

Для enterprise-клиентов VEDO может предоставлять monthly SCI/carbon report. По запросу VEDO помогает выбрать регион с минимальным PUE и высокой долей renewable energy.

Требования по энергоэффективности не являются блокирующими: VEDO Core не потребляет значительных ресурсов. Типовой cluster на 100 пользователей - около 4 vCPU и 16 GB RAM.

## Бизнес-правила

- Sustainability requirements не блокируют MVP, если customer contract явно не требует их выполнения.
- Target для energy per API request: < 0.5 J/request.
- Target для carbon per active user: < 100 g CO2e/month.
- Idle energy consumption должен оставаться < 30% от peak consumption.
- Average cluster CPU utilization должен быть > 40%, чтобы избегать overprovisioning.
- VEDO должен предпочитать Rust и Go для energy-sensitive backend components там, где они подходят задаче.
- VEDO должен избегать busy loops и unnecessary real-time processing.
- Batch processing предпочтителен для analytics, когда real-time не требуется.
- Enterprise deployments могут включать Kepler/Scaphandre dashboards для energy metrics.
- Enterprise customers могут запросить monthly SCI/carbon footprint reports.
- Green cloud region selection рекомендуется, когда это не конфликтует с data residency или customer policy.
- Hardware recycling, e-waste и vendor retirement requirements не входят в MVP sustainability gates.
- Managed private cloud hardware retirement requirements определяются только явным Enterprise/P3 contract scope.

## Открытые вопросы

- Нет открытых вопросов по S1.3.
