### UC-api.webhooks.manage-subscriptions-and-delivery: Управлять webhook-подписками и доставкой событий
**Актор:** Разработчик  
**Приоритет:** P1  
**Ключевая функция:** F6.3 — Webhooks  
**Канал:** API/Webhook  

**Описание:** Настройка webhook-подписок и отправка HTTP-уведомлений при событиях create/update/delete и коммитах.
**Основной поток:**
1. Разработчик создает или обновляет webhook-подписку.
2. Система регистрирует endpoint и параметры доставки.
3. При наступлении события система отправляет POST-уведомление.
4. Система фиксирует результат доставки.
**Постусловия:** Подписка активна, события доставляются по контракту.
**Источник требований:** `human/artifacts/requirements/REQ-FUN.API.integration.md`, `human/artifacts/requirements/REQ-FUN.INTEGRATION.ticket-management.md` (T-18)
