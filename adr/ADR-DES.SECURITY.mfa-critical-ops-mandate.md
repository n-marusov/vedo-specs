# ADR-DES.SECURITY.mfa-critical-ops-mandate

**Дата:** 2026-05-13  
**Статус:** Принято

**Примечание:** ADR дополнен правилами service-account из `human/artifacts/requirements/REQ-FUN.INFRA.vedo-cli-specification.md`. Для CLI-аутентификации спецификация CLI является более специфичным источником и имеет приоритет при конфликте интерпретаций.

**Примечание о согласованности:** Настоящий ADR является авторитетным документом для MFA-политики. Документ `human/artifacts/requirements/REQ-NFR.SECURITY.privileged-access-control.md` детализирует эти правила для привилегированных ролей поддержки и не вводит дополнительных ограничений.

## Контекст

Code Spaces (2014) показал, что компрометация одного набора production credentials может уничтожить и данные, и backup. Для необратимых операций недостаточно одного фактора аутентификации.

## Требование-источник
- [secure-erase.md](requirements/REQ-NFR.DATA.secure-erase.md)
- [backup-retention.md](requirements/REQ-NFR.DATA.backup-retention.md)
- [organization-access-model.md](requirements/REQ-NFR.SECURITY.organization-access-model.md)

## Решение

MFA обязателен и не отключается для категорий A (необратимые destructive: backup delete, tenant purge, secure erase, decommission --purge-data) и B (критические state-changing: production restore, migration rollback); для категории C (стандартные state-changing) MFA включён по умолчанию с возможностью настройки; категория D (read-only) MFA не требует.

Четырёхуровневая категоризация гарантирует, что компрометация одного credentials недостаточна для уничтожения данных или backup — для операций A/B MFA не кэшируется и требует нового подтверждения при каждом вызове. Допустимые вторые факторы: TOTP или WebAuthn/FIDO2; SMS/email не допускаются как единственный второй фактор; recovery codes — как резервный канал.

Для категорий A/B MFA не кэшируется; каждый вызов CLI или API требует нового TOTP/WebAuthn подтверждения; break-glass Emergency Admin с Shamir splitting покрывает отказ IdP.

## Service-account исключение (дополнение к категориям C и D)

Для service-account (автоматизированные сценарии: CI/CD, cron-задачи):

| Категория | Разрешено? | Условия |
|-----------|------------|---------|
| **D** (экспорт, диагностика, backup create) | Да | Без дополнительных ограничений, кроме стандартного RBAC/audit |
| **C** (удаление онтологии, удаление ветки, закрытие аккаунта) | Да | Только с ограниченным scope: явный список разрешенных операций |
| **B** (restore production, migration rollback) | Нет | Требуют MFA + человек |
| **A** (tenant delete, purge, backup delete) | Нет | Требуют MFA + человек + approval |

**Требования к service-account токенам:**

| Параметр | Значение | Обоснование |
|----------|----------|-------------|
| Ограниченный scope | Явный allow-list операций (например, `ontology:delete`, `branch:delete`) | Принцип минимальных привилегий |
| Время жизни (TTL) | <= 1 час (рекомендуемо 15-30 минут для CI/CD) | Снижение риска при компрометации |
| Аудит | Каждое использование токена логируется (кто, когда, какая операция, success/failure) | Compliance и расследования |
| Запрещенные операции | Операции категорий A/B недоступны для service-account | Барьер для наиболее разрушительных действий |

Обоснование: CI/CD не может проходить интерактивный MFA, но операции категории C требуются для автоматизации (очистка тестовых данных, ротация веток, управление онтологиями). Ограниченный scope и короткий TTL выступают компенсирующими мерами.

## MFA для привилегированных ролей поддержки в CLI (дополнение)

В MVP для CLI-аутентификации привилегированных ролей поддержки (`Support Engineer`, `SRE`, `Database Administrator`) разрешен TOTP как основной второй фактор.

Пример CLI-потока:

```bash
$ vedo-cli support tenant-info --id tenant-456
Enter username: support@vedo.cloud
Enter password:
Enter TOTP code: 123456
Authenticated
```

Требования к TOTP в MVP:
- Алгоритм: TOTP (RFC 6238)
- Длина кода: 6 цифр
- Шаг: 30 секунд
- Совместимые приложения: Google Authenticator, FreeOTP, Microsoft Authenticator и другие RFC-совместимые

Почему не WebAuthn/FIDO2 в чистом CLI в MVP:
- WebAuthn опирается на браузерный API (`navigator.credentials`) и не является нативным для чистого CLI.
- Интеграции через внешние утилиты/прокси для hardware tokens усложняют MVP и поддержку в heterogeneous окружениях.

Для Web UI (Support Portal) в MVP также используется TOTP для единообразия. WebAuthn/FIDO2 допускается как опциональное Enterprise-усиление в post-MVP; для CLI это требует веб-компаньон-потока (browser-based confirmation), а не чистого CLI.

Отклоненные варианты для MVP:
- Только пароль: недостаточно по уровню безопасности.
- Обязательный FIDO2/WebAuthn для CLI: технически и операционно избыточно для MVP.
- SMS/email как второй фактор: отклонены из-за повышенного риска компрометации.

## Рассмотренные альтернативы

| Альтернатива | Причина отклонения |
|--------------|--------------------|
| MFA настраивается администратором для всех операций | Позволяет отключить защиту именно там, где риск максимален |
| SMS/email как второй фактор | Уязвимо к SIM-swapping и перехвату почты |
| Кэшировать MFA на CLI-сессию | Один успешный MFA открывает серию destructive-команд |

## Последствия

**Положительные последствия:**
- Компрометация одного credential не достаточна для уничтожения backup или purge tenant.
- Операции A/B имеют архитектурно обязательный security barrier.

**Отрицательные последствия:**
- Экстренные операции могут требовать больше времени.
- Нужно поддерживать on-premise MFA и recovery codes.

**Меры снижения рисков:**
- Break-glass Emergency Admin покрывает отказ IdP, но компенсируется Shamir splitting и audit.
- MFA flow покрывается integration tests и staging drills.

---
