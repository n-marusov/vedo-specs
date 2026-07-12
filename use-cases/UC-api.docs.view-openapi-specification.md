### UC-api.docs.view-openapi-specification: Просматривать OpenAPI/Swagger спецификацию
**Акторы:** Разработчик, Интегратор  
**Приоритет:** P0 (планируется)  
**Ключевая функция:** F6.4 — Swagger / OpenAPI  
**Канал:** API/Portal  

**Описание:** Использование автогенерируемой OpenAPI/Swagger-документации для тестирования и интеграции API.
**Основной поток:**
1. Пользователь открывает API-документацию.
2. Система показывает актуальную OpenAPI-спецификацию.
3. Пользователь проверяет контракты endpoint и примеры запросов.
4. Пользователь запускает тестовый вызов.
**Постусловия:** API-контракт доступен и применим для интеграции.
**Источник требований:** `human/artifacts/requirements/REQ-FUN.API.integration.md`, `human/artifacts/requirements/REQ-NFR.DOC.documentation-training.md`
