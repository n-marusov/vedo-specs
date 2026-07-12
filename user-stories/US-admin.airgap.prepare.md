<a id="us-admin.airgap.prepare"></a>
# US-admin.airgap.prepare: Подготовка air-gapped package

```gherkin
@US-admin.airgap.prepare @UC-admin.airgap.prepare-air-gapped-package-via-cli @P1 @admin @airgap @vedo-cli
Feature: US-admin.airgap.prepare Подготовка air-gapped package

  Background:
    Given DevOps имеет доступ к сборочному окружению VEDO Core

  Scenario: DevOps готовит offline-пакет для изолированной среды
    When DevOps выполняет команду "vedo-cli airgap prepare --output ./dist/airgap"
    Then пакет содержит container images
    And пакет содержит Helm charts
    And пакет содержит offline documentation
    And пакет содержит checksums и default configuration
    When DevOps выполняет команду "vedo-cli airgap verify ./dist/airgap"
    Then проверка подтверждает полноту пакета
    And проверка подтверждает отсутствие runtime-зависимости от public internet
```
