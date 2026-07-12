# Версионирование REST API

| Атрибут | Значение |
|---------|----------|
| **ID** | REQ-FUN.API.rest-versioning |
| **Уровень** | FUN |
| **Атрибут качества** | Interface |
| **Приоритет** | P0 |
| **Статус** | УТВЕРЖДЕНО |
| **Источник** | — |
| **Критерии приёмки** | — |

---

## Подход

REST API версионируется через путь URL с major version: `/api/v1/`, `/api/v2/`.

Для отраслевых интеграций, например VEDO Family и VEDO Agro, при переходе на новую мажорную версию одновременно поддерживаются старая и новая версии API.

## Маршрутизация

API Gateway маршрутизирует разные major версии на соответствующие backend deployments.

```nginx
location /api/v1/ {
    proxy_pass http://backend-v1:8080/;
}

location /api/v2/ {
    proxy_pass http://backend-v2:8080/;
}

location /api/ {
    proxy_pass http://backend-v2:8080/;
}
```

Если версия не указана, `/api/` указывает на latest stable.

## Пример изменения запроса

В v1 клиент создает класс только с `name`:

```http
POST /api/v1/ontologies/123/classes
Content-Type: application/json

{
  "name": "Person"
}
```

В v2 добавлено новое поле `namespace`:

```http
POST /api/v2/ontologies/123/classes
Content-Type: application/json

{
  "name": "Person",
  "namespace": "http://vedo.ai/ontology"
}
```

## Заголовки deprecation

Для deprecated REST version API возвращает предупреждения:

```http
GET /api/v1/classes?limit=10
Warning: 299 - "This API version is deprecated. Please upgrade to /api/v2/. Support ends 2026-06-01"
Sunset: Sun, 01 Jun 2026 23:59:59 GMT
```

## Открытые вопросы

- Нет по M2.2.
