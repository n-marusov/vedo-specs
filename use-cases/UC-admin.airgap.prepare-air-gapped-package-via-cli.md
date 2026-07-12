### UC-admin.airgap.prepare-air-gapped-package-via-cli: Готовить air-gapped package через `vedo-cli`
**Актор:** DevOps  
**Приоритет:** P1 (высокий)  
**Ключевая функция:** F9.3 — Air-gapped подготовка  

**Описание:** Подготовка и проверка offline installation package для изолированной среды.
**Основной поток:**
1. DevOps выполняет `vedo-cli airgap prepare` в connected environment.
2. `vedo-cli` собирает container images, Helm charts, documentation, default configuration и checksums.
3. DevOps переносит package в isolated environment.
4. Выполняет `vedo-cli airgap verify` для проверки полноты пакета и отсутствия runtime internet dependencies.
5. Установка выполняется из local registry / local chart mirror.
**Альтернативные потоки:**
- А1: Отсутствует image или chart — verify завершается ошибкой и показывает недостающий artifact.
- А2: Найдена outbound dependency — package считается неготовым к air-gap.
**Постусловия:** Air-gapped package готов к установке или список недостающих артефактов явно выдан DevOps.
