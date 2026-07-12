### UC-api.auth.issue-and-validate-jwt-tokens: Выдавать и валидировать JWT через Keycloak
**Акторы:** Разработчик, API Gateway  
**Приоритет:** P0 (планируется)  
**Ключевая функция:** F6.2 — Аутентификация  
**Канал:** API  

**Описание:** Получение и валидация JWT-токенов через Keycloak для доступа к API.
**Основной поток:**
1. Клиент получает JWT-токен у провайдера аутентификации.
2. Клиент отправляет API-запрос с Bearer-токеном.
3. API Gateway валидирует токен.
4. Gateway маршрутизирует запрос только после успешной проверки.
**Постусловия:** Доступ к API осуществляется через валидный JWT.
**Источник требований:** `human/artifacts/requirements/REQ-FUN.API.integration.md`, `human/artifacts/requirements/REQ-FUN.API.protocol-stack.md`
