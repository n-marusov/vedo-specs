# ADR-DES.API.protocol-stack-strategy

**Дата:** 2026-05-09  
**Статус:** Принято

## Требование-источник
- [REQ-FUN.API.protocol-stack.md](requirements/REQ-FUN.API.protocol-stack.md)

## Решение

Принять многоуровневый протокольный стек: gRPC для внутренних вызовов между сервисами, REST (JSON) для внешнего API, WebSocket для коллаборации в реальном времени, GraphQL для API навигации по графу клиентской части.

gRCP и protobuf-контракты обеспечивает высокую производительность и строгую типизацию внутренних коммуникаций. REST остаётся стандартом де-факто для внешних интеграций с широкой поддержкой. WebSocket даёт двунаправленную связь с низкой задержкой для совместного редактирования. API Gateway как единый фасад скрывает внутреннюю топологию сервисов и упрощает версионирование и security.

API Gateway преобразует публичные REST/GraphQL/SPARQL команды в gRPC/protobuf вызовы внутренних сервисов через GrpcProxy. Внутренние сервисы не публикуют функциональный REST API, за исключением management surface (`/health`, `/ready`).

## DDoS Mitigation Configuration (API Gateway)

API Gateway реализует многоуровневую защиту от DDoS-атак:

```yaml
# Таймауты (Slowloris, slow body/headers)
client_header_timeout: 5s
client_body_timeout: 10s
proxy_read_timeout: 60s
proxy_connect_timeout: 10s
keepalive_requests: 100

# Ограничение соединений (per IP)
limit_conn_zone $binary_remote_addr zone=conn_per_ip:10m
limit_conn conn_per_ip 10

# Rate limiting per IP
limit_req_zone $binary_remote_addr zone=req_per_ip:10m rate=100r/m
limit_req_zone $http_x_tenant_id zone=req_per_tenant:10m rate=5000r/m

# HTTP/2 Rapid Reset protection
http2_max_concurrent_streams: 128
limit_conn_zone $binary_remote_addr zone=http2_per_ip:10m
limit_conn http2_per_ip 10

# WAF baseline
waf_enabled: true
waf_ruleset: "OWASP CRS"
waf_mode: "block"  # или log-only для калибровки
```

Включено по умолчанию для всех production-окружений. Для SaaS защита L3/L4 обеспечивается cloud-провайдером. On-premise рекомендуется Cloudflare или Nginx с модулем `limit_req/limit_conn`.

---
