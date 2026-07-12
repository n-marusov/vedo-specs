# Network Requirements

| Атрибут | Значение |
|---------|----------|
| **ID** | REQ-CON.INFRA.network-requirements |
| **Уровень** | CON |
| **Атрибут качества** | Physical |
| **Приоритет** | P0 |
| **Статус** | УТВЕРЖДЕНО |
| **Источник** | — |
| **Критерии приёмки** | — |

---

## Клиент → API Gateway

| Параметр | Минимально | Рекомендовано | Метод измерения |
|----------|------------|---------------|-----------------|
| Latency (p95) | ≤ 200 ms | ≤ 50 ms | `vedo-cli diagnose latency` |
| Packet loss | ≤ 1% | ≤ 0.1% | mtr / ping |
| Bandwidth (download) | ≥ 5 Mbps | ≥ 20 Mbps | Speedtest / iperf3 |
| Bandwidth (upload) | ≥ 1 Mbps | ≥ 5 Mbps | Speedtest / iperf3 |
| DNS resolution | < 100 ms | < 50 ms | `dig` |

## Сервис → Сервис (внутренний)

| Параметр | Требование | Примечание |
|----------|------------|------------|
| Latency (gRPC) | ≤ 10 ms (p99) | Между сервисами в одном DC |
| Bandwidth | ≥ 100 Mbps | Для репликации БД, логов |
| Packet loss | ≤ 0.01% | Для стабильности gRPC |

## WebSocket (Commenting Service)

| Параметр | Требование | Примечание |
|----------|------------|------------|
| Latency | ≤ 100 ms | Для приемлемого чата |
| Stability | Поддержание соединения > 1 часа | Без разрывов (keepalive) |

## Firewall requirements (ports)

| Направление | Порт | Протокол | Назначение |
|-------------|------|----------|------------|
| Клиент → API Gateway | 443 | HTTPS | Основной API |
| Клиент → API Gateway | 443 | WSS | WebSocket (комментарии) |
| Сервис → Neo4j (internal) | 7687 | Bolt | gRPC-подобный |
| Сервис → PostgreSQL | 5432 | PostgreSQL | База данных |
| Сервис → Redis | 6379 | RESP | Кэш, locks |
| Сервис → MinIO (S3) | 443 | HTTPS | Объектное хранилище |

**Примечание:** Все внутренние порты должны быть закрыты от внешней сети (кроме API Gateway).
