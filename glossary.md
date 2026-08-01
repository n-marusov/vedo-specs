---
title: Глоссарий по онтологическому моделированию знаний
description: Систематизированный перечень терминов и определений в области семантических технологий, RDF, OWL и методов моделирования знаний, дополненный терминологией платформы VEDO Core.
---

# Глоссарий: Онтологическое моделирование знаний

Этот глоссарий содержит ключевые термины и определения предметной области «Онтологическое моделирование знаний», используемые в Semantic Web, инженерии знаний и при создании экспертных систем. Термины сгруппированы по контекстам для удобства навигации. Включены также термины, относящиеся к платформе **VEDO Core** — многопользовательскому редактору онтологий.

## Оглавление

- [Глоссарий: Онтологическое моделирование знаний](#глоссарий-онтологическое-моделирование-знаний)
  - [Оглавление](#оглавление)
  - [1. Базовые концепты и теория моделирования](#1-базовые-концепты-и-теория-моделирования)
    - [Данные (Data)](#данные-data)
    - [Значение свойства (Value)](#значение-свойства-value)
    - [Индивидуальный объект (Individual)](#индивидуальный-объект-individual)
    - [Интенсионал (Intension)](#интенсионал-intension)
    - [Класс (Class)](#класс-class)
    - [Концептуализация (Conceptualization)](#концептуализация-conceptualization)
    - [Модель (Model)](#модель-model)
    - [Модель данных, схема данных (Data model)](#модель-данных-схема-данных-data-model)
    - [Онтология (Ontology)](#онтология-ontology)
    - [Ссылочное свойство (Property)](#ссылочное-свойство-property)
    - [Свойство-литерал (Datatype Property)](#свойство-литерал-datatype-property)
    - [Таксономия (Taxonomy)](#таксономия-taxonomy)
    - [Триплет (Triple)](#триплет-triple)
    - [Экстенсионал (Extension)](#экстенсионал-extension)
  - [2. Цели и стратегии моделирования](#2-цели-и-стратегии-моделирования)
    - [Декомпозиция (Decomposition)](#декомпозиция-decomposition)
    - [Идентификация (Identification)](#идентификация-identification)
    - [Имитационное моделирование (Simulation Modeling)](#имитационное-моделирование-simulation-modeling)
    - [Классификация (Classification)](#классификация-classification)
    - [Поддержка принятия решений (Decision Support)](#поддержка-принятия-решений-decision-support)
    - [Реификация (Reification)](#реификация-reification)
    - [Фасетная классификация (Faceted Classification)](#фасетная-классификация-faceted-classification)
    - [Цифровой двойник (Digital Twin)](#цифровой-двойник-digital-twin)
  - [3. Структура и архитектура онтологий](#3-структура-и-архитектура-онтологий)
    - [ABox (Assertion Box)](#abox-assertion-box)
    - [TBox (Terminology Box)](#tbox-terminology-box)
    - [RBox (Role Box)](#rbox-role-box)
    - [Дескрипционная логика (Description Logic)](#дескрипционная-логика-description-logic)
    - [Парадигма открытого мира (Open World Assumption)](#парадигма-открытого-мира-open-world-assumption)
  - [4. Технологический стек Semantic Web](#4-технологический-стек-semantic-web)
    - [RDF (Resource Description Framework)](#rdf-resource-description-framework)
    - [RDFS (RDF Schema)](#rdfs-rdf-schema)
    - [OWL (Web Ontology Language)](#owl-web-ontology-language)
    - [SPARQL (SPARQL Protocol and RDF Query Language)](#sparql-sparql-protocol-and-rdf-query-language)
    - [SPARQL-инъекция](#sparql-инъекция)
    - [Параметризованный запрос](#параметризованный-запрос)
    - [SWRL (Semantic Web Rule Language)](#swrl-semantic-web-rule-language)
    - [SHACL (Shapes Constraint Language)](#shacl-shapes-constraint-language)
    - [URI / IRI (Uniform Resource Identifier)](#uri--iri-uniform-resource-identifier)
    - [Triple Store (RDF Store)](#triple-store-rdf-store)
    - [Машина логического вывода (Reasoner)](#машина-логического-вывода-reasoner)
    - [Пустой узел (Blank Node)](#пустой-узел-blank-node)
    - [SPARQL Endpoint](#sparql-endpoint)
    - [Introspection](#introspection)
    - [Query complexity](#query-complexity)
    - [Turtle (Terse RDF Triple Language)](#turtle-terse-rdf-triple-language)
    - [RDF/XML](#rdfxml)
    - [N-Triples](#n-triples)
    - [XSD (XML Schema Definition)](#xsd-xml-schema-definition)
    - [Курсорная пагинация](#курсорная-пагинация)
    - [Глубина запроса (Depth limit)](#глубина-запроса-depth-limit)
    - [Сложность запроса (Query complexity)](#сложность-запроса-query-complexity)
  - [5. Программные инструменты](#5-программные-инструменты)
    - [АрхиГраф (Archigraph)](#архиграф-archigraph)
    - [Fluent Editor](#fluent-editor)
    - [OWL API](#owl-api)
    - [Protégé](#protégé)
    - [TopBraid Composer](#topbraid-composer)
    - [WebProtégé](#webprotégé)
  - [6. Общетехнические термины](#6-общетехнические-термины)
    - [MVC (Model-View-Controller)](#mvc-model-view-controller)
    - [noSQL (Not Only SQL)](#nosql-not-only-sql)
    - [XML (eXtensible Markup Language)](#xml-extensible-markup-language)
    - [XPath / XSLT](#xpath--xslt)
    - [JSON-LD (JSON for Linked Data)](#json-ld-json-for-linked-data)
    - [GraphQL](#graphql)
    - [OpenAPI](#openapi)
    - [Swagger UI](#swagger-ui)
    - [Protocol Buffers / Protobuf](#protocol-buffers--protobuf)
    - [WebSocket](#websocket)
    - [UUID (Universally Unique Identifier)](#uuid-universally-unique-identifier)
    - [JSONB](#jsonb)
  - [7. Термины платформы VEDO Core](#7-термины-платформы-vedo-core)
    - [Платформа VEDO Core (VEDO Core)](#платформа-vedo-core-vedo-core)
      - [VEDO](#vedo)
      - [VEDO Core](#vedo-core)
      - [VEDO Editor](#vedo-editor)
      - [Точка доступа (Point of Access)](#точка-доступа-point-of-access)
      - [Группа (Group)](#группа-group)
      - [Проект (Project)](#проект-project)
    - [Архитектурные компоненты](#архитектурные-компоненты)
      - [Ontology Service](#ontology-service)
      - [Versioning Service](#versioning-service)
      - [API Gateway](#api-gateway)
      - [Auth Service](#auth-service)
      - [Metrics Service](#metrics-service)
      - [Commenting Service](#commenting-service)
      - [Ticket API](#ticket-api)
      - [Ticket Telemetry Listener](#ticket-telemetry-listener)
      - [Ticket Notifier](#ticket-notifier)
      - [Publisher Service](#publisher-service)
      - [Public Browse API](#public-browse-api)
      - [Ticket Classifier](#ticket-classifier)
      - [Frontend](#frontend)
      - [Publish Browse UI](#publish-browse-ui)
      - [Document Extractor](#document-extractor)
      - [vedo-cli](#vedo-cli)
    - [Функции и возможности](#функции-и-возможности)
      - [Каноническая сериализация (Canonical Serialization)](#каноническая-сериализация-canonical-serialization)
      - [Коммит онтологии (Ontology Commit)](#коммит-онтологии-ontology-commit)
      - [Ветка онтологии (Ontology Branch)](#ветка-онтологии-ontology-branch)
      - [Слияние онтологий (Ontology Merge)](#слияние-онтологий-ontology-merge)
      - [Merge Request](#merge-request)
      - [Canonical Workload Profile](#canonical-workload-profile)
      - [Operation Inventory](#operation-inventory)
      - [API Coverage Gate](#api-coverage-gate)
      - [Обратная совместимость API (API Backward Compatibility)](#обратная-совместимость-api-api-backward-compatibility)
      - [Deprecation Period](#deprecation-period)
      - [Долгосрочная поддержка (LTS, Long Term Support)](#долгосрочная-поддержка-lts-long-term-support)
      - [Аварийный выключатель (Kill Switch)](#аварийный-выключатель-kill-switch)
      - [Доказательство развёртывания (Deployment Evidence)](#доказательство-развёртывания-deployment-evidence)
      - [Демо-проект (Demo Project)](#демо-проект-demo-project)
      - [Форк онтологии (Ontology Fork)](#форк-онтологии-ontology-fork)
      - [Snapshot (Release)](#snapshot-release)
      - [Branch Snapshot](#branch-snapshot)
      - [Последовательность построения онтологии (Ontology Build Sequence)](#последовательность-построения-онтологии-ontology-build-sequence)
    - [Интеграция и API](#интеграция-и-api)
      - [REST API VEDO Core](#rest-api-vedo-core)
      - [Webhook онтологии](#webhook-онтологии)
      - [gRPC API](#grpc-api)
    - [Мониторинг и наблюдаемость](#мониторинг-и-наблюдаемость)
      - [OpenTelemetry (OTel)](#opentelemetry-otel)
      - [Трассировка (Trace)](#трассировка-trace)
      - [RED Method](#red-method)
      - [Дашборд (Dashboard)](#дашборд-dashboard)
    - [Фронтенд (VEDO Web UI)](#фронтенд-vedo-web-ui)
      - [Vue 3](#vue-3)
      - [Composition API](#composition-api)
      - [Apollo Client](#apollo-client)
      - [Vue Flow](#vue-flow)
      - [Vite](#vite)
    - [Интерфейс пользователя](#интерфейс-пользователя)
      - [Визуализация графа (Graph Visualization)](#визуализация-графа-graph-visualization)
      - [Дерево классов (Class Tree)](#дерево-классов-class-tree)
      - [Панель свойств (Property Panel)](#панель-свойств-property-panel)
      - [Режим сравнения версий (Diff Mode)](#режим-сравнения-версий-diff-mode)
      - [CTA (Call-to-Action)](#cta-call-to-action)
    - [Инфраструктура и DevOps](#инфраструктура-и-devops)
      - [Канонический Turtle](#канонический-turtle)
      - [LOP (Large Object Promisor)](#lop-large-object-promisor)
      - [Helm Chart](#helm-chart)
      - [MinIO](#minio)
    - [Тип свойства (PropertyType)](#тип-свойства-propertytype)
    - [Аннотационное свойство (Annotation Property)](#аннотационное-свойство-annotation-property)
      - [WORM (Write Once Read Many)](#worm-write-once-read-many)
      - [Support DB](#support-db)
      - [Support Metadata](#support-metadata)
      - [Cobra](#cobra)
    - [Безопасность и управление доступом](#безопасность-и-управление-доступом)
      - [RBAC (Role-Based Access Control)](#rbac-role-based-access-control)
      - [Keycloak](#keycloak)
      - [JWT (JSON Web Token)](#jwt-json-web-token)
      - [OAuth2 / OIDC](#oauth2--oidc)
      - [Identity Provider (IdP)](#identity-provider-idp)
      - [GDPR](#gdpr)
      - [152-ФЗ / ПДн](#152-фз--пдн)
      - [BYOL (Bring Your Own License)](#byol-bring-your-own-license)
      - [Экстренный доступ (Break-glass / Emergency Admin)](#экстренный-доступ-break-glass--emergency-admin)
      - [TOTP (Time-based One-Time Password)](#totp-time-based-one-time-password)
      - [ROPG (Resource Owner Password Grant)](#ropg-resource-owner-password-grant)
      - [Service Account (учётная запись службы)](#service-account-учётная-запись-службы)
      - [M2M-токен (Machine-to-Machine token)](#m2m-токен-machine-to-machine-token)
      - [AuthSessionManager](#authsessionmanager)
      - [Схема разделения секрета Шамира (Shamir Secret Sharing)](#схема-разделения-секрета-шамира-shamir-secret-sharing)
      - [BOLA (Broken Object Level Authorization)](#bola-broken-object-level-authorization)
      - [BFLA (Broken Function Level Authorization)](#bfla-broken-function-level-authorization)
      - [IDOR (Insecure Direct Object Reference)](#idor-insecure-direct-object-reference)
      - [Negative Authorization Test](#negative-authorization-test)
      - [SAST (Static Application Security Testing)](#sast-static-application-security-testing)
      - [DAST (Dynamic Application Security Testing)](#dast-dynamic-application-security-testing)
      - [OWASP ASVS](#owasp-asvs)
      - [SCA (Software Composition Analysis)](#sca-software-composition-analysis)
      - [SBOM (Software Bill of Materials)](#sbom-software-bill-of-materials)
      - [CVE (Common Vulnerabilities and Exposures)](#cve-common-vulnerabilities-and-exposures)
      - [Advisory Database](#advisory-database)
      - [Compliance Evidence](#compliance-evidence)
      - [DDoS (Distributed Denial of Service)](#ddos-distributed-denial-of-service)
      - [WAF (Web Application Firewall)](#waf-web-application-firewall)
      - [Rate Limiting](#rate-limiting)
      - [API Key](#api-key)
      - [Capability Scope](#capability-scope)
      - [Write-Path Invariant](#write-path-invariant)
      - [gitleaks](#gitleaks)
      - [Secret Scanning](#secret-scanning)
      - [Secret Rotation](#secret-rotation)
  - [8. SLA, метрики и восстановление](#8-sla-метрики-и-восстановление)
    - [RTO (Recovery Time Objective)](#rto-recovery-time-objective)
    - [RPO (Recovery Point Objective)](#rpo-recovery-point-objective)
    - [SLA (Service Level Agreement)](#sla-service-level-agreement)
    - [SLO (Service Level Objective)](#slo-service-level-objective)
    - [SLI (Service Level Indicator)](#sli-service-level-indicator)
    - [Error Budget](#error-budget)
    - [P0/P1/P2/P3 Alert Severity](#p0p1p2p3-alert-severity)
    - [Alert Fatigue](#alert-fatigue)
    - [p95 / p99 latency](#p95--p99-latency)
    - [Availability SLO](#availability-slo)
    - [Maintenance Window](#maintenance-window)
    - [GitOps](#gitops)
    - [CI/CD (Continuous Integration / Continuous Deployment)](#cicd-continuous-integration--continuous-deployment)
    - [WAL (Write-Ahead Logging)](#wal-write-ahead-logging)
    - [Откат (Rollback)](#откат-rollback)
    - [Защита окружения (Environment Guard)](#защита-окружения-environment-guard)
    - [Эксплуатационная инструкция (Runbook)](#эксплуатационная-инструкция-runbook)
    - [Incident Commander / Communications Lead / Operations Lead](#incident-commander--communications-lead--operations-lead)
    - [Матрица эскалации (Escalation Matrix)](#матрица-эскалации-escalation-matrix)
    - [Период охлаждения (Cooling-off Period)](#период-охлаждения-cooling-off-period)
    - [Учебная тренировка (Drill)](#учебная-тренировка-drill)
    - [PITR (Point-in-Time Recovery)](#pitr-point-in-time-recovery)
    - [Срок хранения резервных копий (Backup Retention)](#срок-хранения-резервных-копий-backup-retention)
    - [3-2-1 Backup Strategy](#3-2-1-backup-strategy)
    - [Неизменяемая резервная копия (Immutable Backup)](#неизменяемая-резервная-копия-immutable-backup)
    - [Secure Erase](#secure-erase)
    - [Audit Cold Archive](#audit-cold-archive)
    - [ERP (Enterprise Resource Planning)](#erp-enterprise-resource-planning)
    - [PLM (Product Lifecycle Management)](#plm-product-lifecycle-management)
    - [MES (Manufacturing Execution System)](#mes-manufacturing-execution-system)
    - [Geo-репликация](#geo-репликация)
      - [Переключение при отказе (Failover)](#переключение-при-отказе-failover)
    - [Плавная деградация (Graceful Degradation)](#плавная-деградация-graceful-degradation)
    - [Предохранитель (Circuit Breaker)](#предохранитель-circuit-breaker)
      - [Предварительная проверка (Pre-flight Check)](#предварительная-проверка-pre-flight-check)
    - [NPS (Net Promoter Score)](#nps-net-promoter-score)
    - [CES (Customer Effort Score)](#ces-customer-effort-score)
    - [UX-долг (UX Debt)](#ux-долг-ux-debt)
  - [9. Модели развёртывания](#9-модели-развёртывания)
    - [SaaS (Software as a Service)](#saas-software-as-a-service)
    - [On-premise](#on-premise)
    - [Приватное облако (Private Cloud)](#приватное-облако-private-cloud)
    - [Мультитенантность (Multi-tenant)](#мультитенантность-multi-tenant)
    - [IaaS (Infrastructure as a Service)](#iaas-infrastructure-as-a-service)
    - [BYOC (Bring Your Own Cloud)](#byoc-bring-your-own-cloud)
    - [Kubernetes / K8s](#kubernetes--k8s)
    - [Docker](#docker)
    - [Изолированное развёртывание (Air-gapped Deployment)](#изолированное-развёртывание-air-gapped-deployment)
    - [Постепенное обновление (Rolling Update)](#постепенное-обновление-rolling-update)
    - [Сине-зелёное развёртывание (Blue-Green Deployment)](#сине-зелёное-развёртывание-blue-green-deployment)
    - [Канареечный выпуск (Canary Release)](#канареечный-выпуск-canary-release)
    - [HA (High Availability)](#ha-high-availability)
    - [Multi-AZ](#multi-az)
    - [Object Storage / S3-compatible Storage](#object-storage--s3-compatible-storage)
    - [Git LFS (Git Large File Storage)](#git-lfs-git-large-file-storage)
    - [Local Registry](#local-registry)
    - [Helm Chart](#helm-chart-1)
    - [Feature Flags (Feature Toggles)](#feature-flags-feature-toggles)
    - [Теневое развёртывание (Shadow Mode)](#теневое-развёртывание-shadow-mode)
    - [CapEx (Capital Expenditure)](#capex-capital-expenditure)
    - [OpEx (Operational Expenditure)](#opex-operational-expenditure)
    - [ФСТЭК](#фстэк)
    - [WCAG (Web Content Accessibility Guidelines)](#wcag-web-content-accessibility-guidelines)
    - [ARIA (Accessible Rich Internet Applications)](#aria-accessible-rich-internet-applications)
    - [Screen Reader](#screen-reader)
    - [Рефлоу (Reflow)](#рефлоу-reflow)
  - [10. Методология, архитектура и качество](#10-методология-архитектура-и-качество)
    - [ADR Naming Convention](#adr-naming-convention)
    - [ADR (Architecture Decision Record)](#adr-architecture-decision-record)
    - [C4 Model](#c4-model)
    - [NFR (Non-Functional Requirement)](#nfr-non-functional-requirement)
    - [PBT (Property-Based Testing)](#pbt-property-based-testing)
    - [Gate](#gate)
    - [Фаззинг-тестирование (Fuzz Testing)](#фаззинг-тестирование-fuzz-testing)
    - [Базовая линия производительности (Performance Baseline)](#базовая-линия-производительности-performance-baseline)
    - [Защита во время выполнения (Runtime Protection)](#защита-во-время-выполнения-runtime-protection)
    - [Усечение результата (Result Truncation)](#усечение-результата-result-truncation)
    - [Traceability](#traceability)
    - [HLV (Human-Led Validation)](#hlv-human-led-validation)
    - [Saga Pattern](#saga-pattern)
    - [Пробный прогон (Dry-run)](#пробный-прогон-dry-run)
    - [Чтение после записи (Read-after-write)](#чтение-после-записи-read-after-write)
    - [Углеродный след (Carbon Footprint)](#углеродный-след-carbon-footprint)

---

## 1. Базовые концепты и теория моделирования

### Данные (Data)
`data`

Совокупность конкретных индивидуальных объектов и значений их свойств в рамках модели данных.

### Значение свойства (Value)
`value`

Значение литерального свойства, выраженное определённым типом данных (строковым, числовым, логическим и т. д.) «Значением» ссылочного свойства может считаться другой индивидуальный объект.

### Индивидуальный объект (Individual)
`individual`

Конкретный объект, элемент модели, представляющий собой отдельную сущность реального мира (или воображаемого). В отличие от класса, индивид не может содержать другие экземпляры (является «атомом» модели в рамках заданного контекста).

### Интенсионал (Intension)
`intension`

Содержание понятия, совокупность существенных признаков, свойств и внутренней идеи, которая определяет класс. Интенсионал отвечает на вопрос: «Что означает это понятие?». Один и тот же экстенсионал (набор объектов) может соответствовать разным интенсионалам.

### Класс (Class)
`class`

Символ или понятие, используемое для обозначения совокупности объектов, обладающих общими свойствами (экстенсионала) и объединенных общей идеей (интенсионалом). Классы могут образовывать иерархии (таксономии).

### Концептуализация (Conceptualization)
`conceptualization`

Абстрактное, упрощенное представление фрагмента реальности, выраженное через набор понятий (концептов), описывающих объекты, явления, свойства и отношения между ними. Является промежуточным звеном между реальным миром и формальной онтологией.

### Модель (Model)
`model`

Информационное представление совокупности объектов и явлений, характеризующееся упрощением (фиксация только существенных черт), концептуализированностью (выражение через понятия) и прогностическим потенциалом (возможность делать выводы).

### Модель данных, схема данных (Data model)
`dataModel`

Модель, задаваемая совокупностью классов и свойств.

### Онтология (Ontology)
`ontology`

Формальная, явная спецификация разделяемой концептуализации. В инженерии знаний — это структура данных, содержащая классы, индивиды, свойства, отношения и аксиомы, описывающие семантику предметной области в машиночитаемом виде (обычно на основе RDF/OWL).

### Ссылочное свойство (Property)
`property`

Характеристика объекта, которая может быть выражена либо значением-литералом (число, строка), либо связью с другим объектом. Свойства в RDF/OWL существуют независимо от классов (глобально) и могут иметь ограничения (домен, диапазон).

### Свойство-литерал (Datatype Property)
`datatypeProperty`

Тип свойства, значением которого является конкретное данное (литерал): число, строка, дата, логическое значение и т.д. Связывает индивид со значением примитивного типа XML Schema (XSD).

### Таксономия (Taxonomy)
`taxonomy`

Иерархическая структура организации классов (понятий), основанная на отношении обобщения/специализации (отношение «is-a», «надкласс-подкласс»).

### Триплет (Triple)
`triple`

Фундаментальная структурная единица RDF, представляющая собой элементарное высказывание в форме {Субъект, Предикат, Объект} (аналог подлежащего, сказуемого и дополнения). Лежит в основе семантических графов.

### Экстенсионал (Extension)
`extension`

Объем понятия, т.е. множество всех объектов реального или воображаемого мира, которые подпадают под данное понятие (класс). Отвечает на вопрос: «Какие объекты относятся к этому классу?».

## 2. Цели и стратегии моделирования

### Декомпозиция (Decomposition)
`decomposition`

Процесс разделения моделируемого фрагмента реальности на отдельные элементы (объекты), которые становятся базовыми единицами информационной модели. Глубина декомпозиции определяется прагматикой (целями моделирования).

### Идентификация (Identification)
`identification`

Процесс присвоения объекту модели уникального обозначения (идентификатора). Ключевой принцип — нейтральность идентификатора (он не должен нести смысла и основываться на изменяемых свойствах объекта).

### Имитационное моделирование (Simulation Modeling)
`simulationModeling`

Метод, при котором модель воспроизводит поведение сложной системы во времени. Элементы модели взаимодействуют по заданным правилам, что позволяет прогнозировать состояние системы при различных сценариях.

### Классификация (Classification)
`classification`

Логическая операция отнесения объекта к одному или нескольким классам на основе заданных признаков. В онтологиях поддерживается множественная классификация (фасетный принцип).

### Поддержка принятия решений (Decision Support)
`decisionSupport`

Применение онтологий и логического вывода для предоставления лицу, принимающему решение, обоснованных вариантов действий, прогнозов последствий и нормативных обоснований. Система не заменяет волевой акт субъекта, но снабжает его информацией.

### Реификация (Reification)
`reification`

Технический прием «разворачивания» связи (ребра графа) в отдельный узел (объект). Позволяет хранить дополнительную метаинформацию об отношении (например, период времени действия связи, степень уверенности).

### Фасетная классификация (Faceted Classification)
`facetedClassification`

Принцип классификации, при котором объект может быть одновременно отнесен к нескольким независимым классификационным группировкам (фасетам). В отличие от строгой иерархии (дерева), допускает множественное наследование свойств.

### Цифровой двойник (Digital Twin)
`digitalTwin`

Интегрированная модель подконтрольной системы (физического объекта, процесса или услуды), которая полностью описывает его структуру, состояние, поведение и взаимодействие с внешней средой в цифровом виде.

## 3. Структура и архитектура онтологий

### ABox (Assertion Box)
`abox`

Часть онтологии, содержащая утверждения о конкретных индивидуальных объектах (индивидах). Хранит фактографические данные (экземпляры классов и их конкретные значения свойств).

### TBox (Terminology Box)
`tbox`

Часть онтологии, содержащая «терминологию» — определения классов, свойств (объектных и дататайп), отношений и общих ограничений (аксиом). Задает схему или структуру знаний предметной области.

### RBox (Role Box)
`rbox`

Компонент онтологии (в терминах дескрипционных логик), содержащий аксиомы и свойства, относящиеся исключительно к ролям (объектным свойствам): иерархию ролей, характеристики (транзитивность, симметричность, инверсность) и цепочки свойств.

### Дескрипционная логика (Description Logic)
`descriptionLogic`

Семейство формальных языков представления знаний (подмножество логики предикатов первого порядка), лежащих в основе OWL. Характеризуется разрешимостью (гарантированное завершение вычислений) и хорошими вычислительными свойствами.

### Парадигма открытого мира (Open World Assumption)
`openWorldAssumption`

Допущение (принятое в OWL и логическом выводе), согласно которому отсутствие факта в онтологии не означает его ложности. Информация считается неполной, и машина вывода не может делать вывод «от противного» на основе отсутствия данных.

## 4. Технологический стек Semantic Web

### RDF (Resource Description Framework)
`rdf`

Стандарт W3C для описания ресурсов в виде графа. Модель данных основана на триплетах (субъект, предикат, объект). Обеспечивает универсальный способ представления метаданных и связей между любыми сущностями в сети.

### RDFS (RDF Schema)
`rdfs`

Стандартный словарь (онтология) для RDF, вводящий базовые понятия моделирования: классы (`rdfs:Class`), свойства (`rdf:Property`), иерархию классов (`rdfs:subClassOf`), домен/диапазон свойств (`rdfs:domain`, `rdfs:range`).

### OWL (Web Ontology Language)
`owl`

Язык описания онтологий, основанный на дескрипционных логиках. Расширяет RDF/RDFS, позволяя задавать сложные аксиомы: эквивалентность и непересекаемость классов, кардинальность свойств, свойства транзитивности/симметричности.

### SPARQL (SPARQL Protocol and RDF Query Language)
`sparql`

Язык запросов к RDF-графам и протокол передачи таких запросов. Позволяет извлекать, добавлять и удалять триплеты, соответствующие заданным графовым шаблонам. Аналог SQL для реляционных баз данных. В VEDO Core SPARQL 1.1 используется как основной язык семантического поиска; производственный endpoint выполняет только формы чтения SELECT/ASK.

### SPARQL-инъекция
`sparqlInjection`

Атака, при которой злоумышленник внедряет код в параметры SPARQL-запроса, изменяя его семантику. В VEDO Core предотвращается параметризованными запросами и строгим экранированием значений, если конкретная библиотека не поддерживает полноценную параметризацию SPARQL.

### Параметризованный запрос
`parameterizedQuery`

Техника, при которой значения параметров передаются отдельно от структуры запроса, исключая возможность инъекции. Аналог prepared statements в SQL. Для SPARQL применяется к значениям фильтров, литералам и IRI, которые приходят из GUI или внешнего API-клиента.

### SWRL (Semantic Web Rule Language)
`swrl`

Язык правил логического вывода для OWL онтологий. Позволяет определять правила вида «Если (условия), То (следствие)» для генерации новых знаний на основе существующих фактов.

### SHACL (Shapes Constraint Language)
`shacl`

Язык W3C для валидации RDF-графов. Позволяет описывать «формы» (ожидаемую структуру данных), ограничения на значения свойств и кардинальность. Используется для проверки качества данных в RDF хранилищах.

### URI / IRI (Uniform Resource Identifier)
`uri`

Унифицированный идентификатор ресурса. В Semantic Web используется для глобальной уникальной идентификации всех сущностей: классов, свойств, индивидов. Может выглядеть как URL (с протоколом http), но не обязан указывать на реально существующий сайт.

### Triple Store (RDF Store)
`tripleStore`

Специализированная база данных (графовая СУБД) для хранения, обработки и выполнения запросов к RDF-графам. Поддерживает индексацию по субъекту, предикату и объекту. Является аналогом реляционной СУБД для семантических технологий.

### Машина логического вывода (Reasoner)
`reasoner`

Программный компонент, предназначенный для вычисления логических следствий из аксиом и фактов онтологии. Проверяет консистентность (непротиворечивость) модели, автоматически выводит принадлежность индивидов к классам и иерархические связи на основе формальной семантики OWL.

### Пустой узел (Blank Node)
`blankNode`

Узел в RDF-графе, не имеющий глобального URI-идентификатора. Используется для описания анонимных ресурсов или сложных структур (списков, ограничений), когда введение отдельного идентификатора нецелесообразно.

### SPARQL Endpoint
`sparqlEndpoint`

Сетевой интерфейс доступа к RDF-хранилищу (Triple Store), принимающий SPARQL-запросы через HTTP и возвращающий результаты в стандартном формате (JSON, XML, CSV, RDF). В VEDO Core endpoint `/api/v1/sparql` должен быть аутентифицированным, доступным только для чтения, защищённым от инъекций и ограниченным по времени выполнения, объёму результатов, частоте запросов и сложности запроса.

### Introspection
`introspection`

Возможность клиента получить метаданные текущей онтологии: список классов, свойств, типов данных и применимых связей. Используется для динамического построения интерфейса конструктора запросов, потому что пользовательский RDF-граф может быть произвольным и не имеет заранее фиксированной схемы.

### Query complexity
`queryComplexity`

Оценка ресурсоёмкости запроса. Для GraphQL учитываются веса полей, вложенность и размер запрашиваемых страниц. Для SPARQL учитываются triple patterns, вложенность, фильтры, `OPTIONAL`, `UNION`, `FILTER NOT EXISTS` и другие дорогостоящие конструкции. Превышение лимита блокирует выполнение запроса до обращения к хранилищу.

### Turtle (Terse RDF Triple Language)
`turtle`

Текстовый формат сериализации RDF-графов, более компактный и читаемый, чем RDF/XML. В VEDO Core используется для импорта, экспорта, канонической сериализации и стабильного diff.

### RDF/XML
`rdfXml`

XML-сериализация RDF-графа. Используется как совместимый формат импорта/экспорта для старых RDF/OWL-инструментов и десктопных редакторов онтологий.

### N-Triples
`nTriples`

Построчный формат сериализации RDF, где каждая строка содержит один RDF-триплет. Удобен для потоковой обработки, валидации и обмена большими RDF-дампами.

### XSD (XML Schema Definition)
`xsd`

Набор стандартных типов данных XML Schema. В OWL/RDF используется для значений DatatypeProperty, например `xsd:string`, `xsd:double`, `xsd:dateTime`.

### Курсорная пагинация
`cursorPagination`

Механизм пагинации, при котором клиент передаёт непрозрачный курсор — строку, указывающую на позицию в списке. Исключает проблемы пропуска и дублирования при параллельных изменениях, характерные для offset-пагинации. В GraphQL-навигации VEDO Core используется через `first`, `after`, `endCursor` и `hasNextPage`.

### Глубина запроса (Depth limit)
`depthLimit`

Максимальная допустимая вложенность полей в навигационном запросе. Используется для предотвращения атак через рекурсивные или чрезмерно вложенные GraphQL-запросы.

### Сложность запроса (Query complexity)
`graphqlQueryComplexity`

Численная оценка ресурсоёмкости GraphQL-запроса. Каждому полю присваивается вес, а суммарная сложность с учётом вложенности и размера страниц не должна превышать установленный порог.

## 5. Программные инструменты

### АрхиГраф (Archigraph)
`archigraph`

Семейство российских программных продуктов компании «ТриниДата» для работы с онтологиями. Включает редактор онтологий (АрхиГраф.Мир), систему управления знаниями (АрхиГраф.СУЗ) и MDM-платформу.

### Fluent Editor
`fluentEditor`

Редактор онтологий компании Cognitum, поддерживающий вход на контролируемом естественном языке. Позволяет формализовать знания на подмножестве английского или русского языка с автоматической трансляцией в OWL.

### OWL API
`owlApi`

Программный интерфейс (Java-библиотека) для работы с OWL-онтологиями. Предоставляет высокоуровневый доступ к структурам RDF/OWL, чтению/записи онтологий, взаимодействию с Reasoner'ами и Triple Store.

### Protégé
`protege`

Свободно распространяемый десктопный редактор онтологий с открытым исходным кодом (Java). Разработан Стэнфордским университетом. Поддерживает все современные стандарты OWL, имеет архитектуру плагинов (Reasoner, визуализация, SWRL).

### TopBraid Composer
`topbraidComposer`

Коммерческий редактор онтологий и инструмент для разработки приложений на основе семантических технологий. Ориентирован на корпоративное использование, поддержку SHACL, SWRL и интеграцию с графовыми базами данных.

### WebProtégé
`webprotege`

Веб-ориентированная версия редактора Protégé, предоставляющая возможности совместной работы над онтологией через браузер. Облачное решение для командной работы без установки тяжелого десктопного ПО.

## 6. Общетехнические термины

### MVC (Model-View-Controller)
`mvc`

Архитектурный паттерн проектирования приложений, разделяющий данные (Model), пользовательский интерфейс (View) и логику управления (Controller). Упоминается в глоссарии для контраста с модель-управляемыми системами, где структура данных сама определяет логику обработки.

### noSQL (Not Only SQL)
`nosql`

Класс баз данных, не использующих реляционную модель и язык SQL. Включает документо-ориентированные (MongoDB), графовые (Neo4j, Triple Store), колоночные и ключ-значение хранилища.

### XML (eXtensible Markup Language)
`xml`

Расширяемый язык разметки, используемый для сериализации данных и документов. Применяется в RDF/XML синтаксисе для представления RDF-графов и в конфигурационных файлах.

### XPath / XSLT
`xpath` / `xslt`

Стандарты W3C для навигации по XML-документу (XPath) и трансформации XML-документов в другие форматы (XSLT). Актуальны при извлечении данных из устаревших систем для интеграции с онтологиями.

### JSON-LD (JSON for Linked Data)
`jsonLd`

Формат сериализации связанных данных (Linked Data) на основе легковесного формата JSON. Позволяет представить RDF-граф в виде, удобном для JavaScript-приложений и веб-разработчиков.

### GraphQL
`graphql`

Язык запросов к API, позволяющий клиенту явно запрашивать нужную структуру данных. В VEDO Core используется как основной API фронтенда для навигации по графу онтологии: подгрузки порций графа, cursor pagination (Relay Connections), вложенных запросов и работы с Apollo Client. GraphQL — строго read-only (13 резолверов навигации); все мутации, версионирование, орг-модель и SPARQL вынесены в REST. Динамические свойства онтологии возвращаются через типизированные поля `literalValues` (свойства-литералы) и `referenceValues` (ссылочные свойства) на типе `Individual`, а навигация по связям графа — через запрос `graphNeighborhood`. Пользовательские свойства не регистрируются как статические поля схемы.

### OpenAPI
`openApi`

Машиночитаемая спецификация REST API. В VEDO Core используется для публикации документации API и автоматической проверки, что endpoint'ы из Operation Inventory реально существуют.

### Swagger UI
`swaggerUi`

Веб-интерфейс для просмотра и ручного вызова REST API на основе OpenAPI-спецификации. Используется для документации и проверки доступности API.

### Protocol Buffers / Protobuf
`protobuf`

Бинарный формат описания сообщений и сервисных контрактов. Используется вместе с gRPC для внутренних API и требует строгой совместимости при изменении `.proto`-схем.

### WebSocket
`webSocket`

Двунаправленный протокол связи поверх HTTP. В VEDO Core используется для уведомлений о событиях версионирования и комментариях к Merge Request'ам (опционально, post-MVP, через API Gateway).

### UUID (Universally Unique Identifier)
`uuid`

Универсальный уникальный идентификатор. Используется для устойчивой адресации онтологий, коммитов, Merge Request'ов и других сущностей API.

### JSONB
`jsonb`

Бинарный формат хранения JSON в PostgreSQL. В VEDO Core применяется для хранения дельт коммитов, метаданных и структурированных payload'ов, требующих индексации и транзакционности.

## 7. Термины платформы VEDO Core

В данном разделе представлены термины, относящиеся к платформе VEDO Core — многопользовательскому веб-редактору онтологий с поддержкой Git-подобного версионирования и API-интеграций.

### Платформа VEDO Core (VEDO Core)

#### VEDO
`vedo`

**Virtual Environment for Developing Ontologies** (Виртуальная среда для разработки онтологий) — акроним платформы. Отражает амбицию VEDO как экосистемы, а не просто редактора: среды, в которой живут онтологии, инженеры знаний, сообщество и интеграции.

Уточняющий слоган: *Visual Editor for Distributed Ontologies* (Визуальный редактор для распределенных онтологий). Используется как подзаголовок на лендинге или для обозначения конкретного продукта: **VEDO Editor**.

Дескриптор: *Build, connect, and share knowledge at scale.*

#### VEDO Core
`vedoCore`

VEDO («Virtual Environment for Developing Ontologies» — виртуальная среда для разработки онтологий) — веб-ориентированная платформа для многопользовательской разработки и редактирования онтологий (графов знаний). Обеспечивает высокую производительность, версионирование (аналог Git) с параллельной работой через ветки и API для интеграции с внешними системами. Является альтернативой десктопным редакторам (Protégé, TopBraid Composer) и веб-инструментам (WebProtégé).

#### VEDO Editor
`vedoEditor`

Конкретное применение акронима VEDO: *Visual Editor for Distributed Ontologies* — визуальный редактор для распределенных онтологий. Используется для обозначения фронтенд-части VEDO Core (VEDO Web UI) в контексте брендинга.

#### Точка доступа (Point of Access)
`accessPoint`

В рамках VEDO Core — изолированное пространство данных, содержащее один или несколько проектов, в каждом из которых разрабатывается онтология. Служит единицей адресации при работе через API и разграничения прав доступа для команд. Project и Ontology — разные сущности: Project — платформенный контейнер для совместной работы, Ontology — его графовое содержимое (см. отдельные записи).

#### Группа (Group)
`group`

Контейнер для объединения проектов и вложенных групп, соответствующий организационной структуре команд, департаментов или направлений. Поддерживает иерархию: группа может содержать другие группы. Членство и права доступа наследуются по иерархии: роль, назначенная на группу, применяется ко всем вложенным группам и проектам. Аналог Group в GitLab. Группа не содержит онтологии напрямую — онтологии живут внутри проектов, а проекты объединяются группами.

#### Проект (Project)
`project`

Платформенная сущность-контейнер для совместной работы над онтологией в VEDO Core. Базовая единица работы, доступа и независимого версионирования. Каждый проект имеет собственный Git-подобный репозиторий (история коммитов, ветки, Merge Request'ы), независимый набор участников с ролями, уровень видимости (visibility) и атрибутные политики доступа. Аналог Project в GitLab.

Project ≠ Ontology. Project — это среда (workspace) для совместной разработки: участники, роли, ветки, merge requests, visibility, policies. Ontology — содержимое проекта: формальная спецификация концептуализации (классы, свойства, индивиды, аксиомы, TBox/ABox), см. запись «Онтология (Ontology)» в разделе 1. Членство (members), видимость (visibility) и политики (policies) относятся к Project, не к Ontology.

Соответствие 1:1: один Project содержит ровно одну Ontology. Это соответствует GitLab-модели «один репозиторий = один Project». Группировка нескольких онтологий выполняется через Group (иерархия групп), а не через упаковку нескольких онтологий в один Project. Project без Ontology не существует; Ontology вне Project не существует.

Проект может иметь опциональное поле `upstream_project_id` — ссылка на другой Project (источник форка). Если поле не null, данный Project является форком указанного upstream-проекта (аналог `forked_from` в GitLab). `upstream_project_id = null` для оригинальных (не-форкнутых) проектов. При удалении upstream-проекта поле сбрасывается в null (orphan fork), но fork продолжает существовать как независимый проект.

### Архитектурные компоненты

#### Ontology Service
`ontologyService`

Микросервис на Rust, являющийся ядром VEDO Core. Обеспечивает выполнение CRUD-операций над узлами и связями графа онтологии, валидацию OWL-конструкций и координацию вызовов к хранилищу данных (Triple Store).

#### Versioning Service
`versioningService`

Микросервис (Rust), реализующий Git-подобное управление версиями для онтологий. Хранит историю коммитов, управляет ветками, выполняет слияние изменений и сравнение версий (diff) на семантическом уровне (а не на уровне строк файла). Также управляет Merge Request'ами — процессом ревью и слияния изменений между ветками.

#### API Gateway
`apiGateway`

Единая точка входа для всех клиентских запросов (веб-интерфейс, внешние API-клиенты). Реализует маршрутизацию, аутентификацию (JWT, Keycloak), ограничение частоты запросов (rate limiting) и сбор метрик.

#### Auth Service
`authService`

Микросервис аутентификации и авторизации. Отвечает за проверку JWT, интеграцию с Keycloak/OAuth/OIDC, emergency fallback и применение ролей доступа.

#### Metrics Service
`metricsService`

Микросервис сбора и публикации продуктовых, системных и performance-метрик VEDO Core. Используется для аналитики, SLO-дашбордов и observability.

#### Commenting Service
`commentingService`

Микросервис коллаборации, отвечающий за комментарии к изменениям онтологии и связанным workflow-сущностям. Обеспечивает создание, чтение и управление потоками обсуждений для совместной работы команды.

#### Ticket API
`ticketApi`

Сервис API для работы с тикетами поддержки и операционными запросами: создание, изменение статусов, назначение ответственных и получение карточек тикетов. Используется как точка интеграции с внешними трекерами и внутренними процессами поддержки.

#### Ticket Telemetry Listener
`ticketTelemetryListener`

Сервис-подписчик, который принимает телеметрию и сервисные события, влияющие на SLA/инциденты, и преобразует их в сигналы для тикетного контура. Нужен для автоматизации реакции на события эксплуатации.

#### Ticket Notifier
`ticketNotifier`

Сервис уведомлений тикетного контура. Рассылает изменения по тикетам (создание, эскалация, смена статуса, закрытие) в каналы оповещения и внешние интеграции.

#### Publisher Service
`publisherService`

Микросервис публикации и экспорта онтологий. Отвечает за формирование snapshot-артефактов (releases) и подготовку данных для внешнего распространения/потребления. Реализует CQRS-модель: снимок = материализованный read-only serving слой (отдельное read-only хранилище — второй Neo4j или MinIO+index, решение отложено). Поддерживает бинарную модель доступа: **public** (без аутентификации) / **restricted** (API key через заголовок `X-VEDO-API-Key`). Publish gate: роль Maintainer+ (вес ≥ 2), отдельной роли publisher нет. **Текущая реализация — in-memory stubs**.

#### Public Browse API
`publicBrowseApi`

Публичный read-only API для просмотра опубликованных онтологий (releases) и связанных метаданных без доступа к операциям редактирования. Используется для сценариев внешнего поиска, навигации и чтения. Поддерживает бинарную модель доступа: **public** (без auth, базовая защита от DoS) / **restricted** (API key через `X-VEDO-API-Key`). Rate limiting per-IP и per-API-key. Visibility снэпшота декоррелирована от visibility Project. **Текущая реализация — in-memory stubs**.

#### Ticket Classifier
`ticketClassifier`

Сервис классификации тикетов, который автоматически определяет тип, приоритет и маршрутизацию обращения по входным данным и контексту. Поддерживает ускорение triage и снижение ручной нагрузки на поддержку.

#### Frontend
`frontend`

Клиентское веб-приложение VEDO Core (основной UI для авторизованных пользователей), через которое выполняются операции редактирования онтологий, навигация по графу, управление версиями и совместная работа. Реализовано на TypeScript с использованием Vue 3 и Vite.

#### Publish Browse UI
`publishBrowseUi`

Отдельный веб-интерфейс для просмотра опубликованных данных в режиме чтения. Предназначен для внешних пользователей и сценариев публичного/партнёрского доступа без функций редактирования, работает в связке с `Public Browse API`.

#### Document Extractor
`documentExtractor`

Микросервис на Python для извлечения OWL-онтологии из загружаемых документов (MD, TXT, PDF, DOCX, JSON, XML, CSV, XLSX). Определяет формат файла, извлекает текст и структуру с учётом форматирования, передаёт содержимое LLM-провайдеру для генерации последовательности операций построения онтологии, проверяет корректность результата и показывает предпросмотр пользователю. После подтверждения передаёт последовательность в сервис онтологий для атомарного выполнения в единой транзакции Neo4j.

#### vedo-cli
`vedoCli`

Административная утилита командной строки для экосистемы VEDO Core. Реализована на Go (single binary) с использованием фреймворка Cobra. Предназначена для эксплуатационных операций: backup/restore, миграции, диагностика инцидентов, air-gapped подготовка, управление tenant и тикетами. Не заменяет web UI и public API — зона ответственности ограничена операциями, требующими автоматизации, повышенных привилегий или работы в air-gapped окружениях. Решения зафиксированы в `ADR-IMPL.STACK.vedo-cli-language-strategy`, `ADR-IMPL.STACK.vedo-cli-framework-strategy` и `ADR-DES.SECURITY.cli-mfa-strategy`.

### Функции и возможности

#### Каноническая сериализация (Canonical Serialization)
`canonicalSerialization`

Процесс приведения RDF-графа (онтологии) к единому, детерминированному текстовому представлению (Turtle). Применяется для обеспечения стабильности Git-диффа: изменения в онтологии отражаются только как изменённые строки, без «шума» от перестановки триплетов.

#### Коммит онтологии (Ontology Commit)
`ontologyCommit`

Фиксация изменений в онтологии (аналог `git commit`). Фиксирует дельту изменений на уровне семантических сущностей (добавление/удаление классов, свойств, индивидов), а не на уровне строк файла. Содержит сообщение, автора, временную метку и ссылку на родительский коммит.

#### Ветка онтологии (Ontology Branch)
`ontologyBranch`

Независимая линия разработки онтологии (аналог `git branch`). Позволяет нескольким разработчикам или группам параллельно вносить изменения без влияния на основную версию (`main`).

#### Слияние онтологий (Ontology Merge)
`ontologyMerge`

Процесс объединения изменений из одной ветки в другую (аналог `git merge`). Выполняется через Merge Request: автор создаёт MR, назначенный ревьюер проверяет Semantic Diff, утверждает или отклоняет изменения. Включает автоматическое разрешение конфликтов на основе семантики (например, если в обеих ветках изменён один и тот же класс) и создание коммита слияния.

#### Merge Request
`mergeRequest`

Процесс контролируемого внесения изменений в защищённую ветку (например, `main`). Включает ветку-источник, ветку-назначения, Semantic Diff (визуальное сравнение версий онтологии), обсуждение и ревью. Статусы: `draft` → `open` → `approved` → `merged` / `closed`. Является основным механизмом параллельной работы: несколько пользователей работают в разных ветках, изменения объединяются через Merge Request после ревью.

#### Canonical Workload Profile
`canonicalWorkloadProfile`

Канонический профиль нагрузки VEDO Core: 1M аксиом, примерно 4M триплетов, 50K классов, 100K индивидов, 5 параллельных редакторов и 10 API requests/sec. Используется для SLA, capacity planning, performance benchmarks и лицензирования.

#### Operation Inventory
`operationInventory`

Поименованный перечень всех UI-операций с указанием места в интерфейсе, эквивалентного API endpoint, параметров и тестового покрытия. Используется для доказательства, что UI-функции доступны через API.

#### API Coverage Gate
`apiCoverageGate`

Автоматическая проверка, которая сравнивает Operation Inventory с OpenAPI-спецификацией и блокирует Merge Request, если покрытие UI-операций через API ниже 100%.

#### Обратная совместимость API (API Backward Compatibility)
`apiBackwardCompatibility`

Свойство новой версии API сохранять работоспособность клиентов, написанных под предыдущую версию. Для VEDO Core особенно важно для on-premise и LTS-клиентов.

#### Deprecation Period
`deprecationPeriod`

Период предупреждения, в течение которого устаревшая версия API, поле или операция ещё поддерживаются перед удалением. Нужен для безопасной миграции клиентов.

#### Долгосрочная поддержка (LTS, Long Term Support)
`lts`

Версия с длительным сроком сопровождения. В VEDO Core ориентирована на корпоративных и on-premise-клиентов, которым требуются стабильный API, предсказуемые обновления и длительный период получения исправлений.

#### Аварийный выключатель (Kill Switch)
`killSwitch`

Логический механизм экстренного перевода системы в режим «только чтение» без остановки сервиса. Активируется командой `vedo-cli emergency readonly`; API Gateway отклоняет все изменяющие запросы (POST/PUT/PATCH/DELETE) с кодом HTTP 503 и заголовком `X-VEDO-Emergency: readonly`, а запросы чтения продолжает обслуживать. Время срабатывания — не более 60 секунд. Механизм доступен через независимый управляющий endpoint с приоритетным путём выполнения. Физическое отключение через сетевые правила или остановку сервиса используется только как резервный вариант, если логический механизм недоступен.

#### Доказательство развёртывания (Deployment Evidence)
`deploymentEvidence`

Автоматически фиксируемый набор данных о каждом производственном развёртывании: идентификатор развёртывания, время начала и завершения, инициатор, ссылка на артефакт (SHA коммита, тег образа), целевое окружение, статус проверки и ссылка на предыдущую версию для отката. Формируется CI/CD-пайплайном без возможности ручного пропуска или последующего заполнения. Хранится отдельно от производственной среды в неизменяемом хранилище (S3 Object Lock / WORM), в режиме append-only, с SHA-256-подписью. Используется как аудиторский след и основа для расследования инцидентов.

#### Демо-проект (Demo Project)
`demoProject`

Публичный Project в группе `VEDO Demos`, используемый как отправная точка при создании новой онтологии через fork. Каждый демо-проект содержит валидную OWL-онтологию с TBox (классы, свойства, иерархия, аксиомы), минимальным ABox (1–2 демонстрационных индивида) и документацией. В отличие от прежней концепции «Шаблон онтологии» (Ontology Template, удалена), демо-проект не имеет отдельного semver-версионирования, metadata.json, usage_count или жизненного цикла review/deprecate — вместо этого используется git-теги для версионирования и forks_count как метрика популярности. Набор из 5 демо-проектов (Организация, Продукт, Процесс, Глоссарий, Событие) создаётся как seed data при деплое через `deploy/seeds/vedo-demos/bootstrap.sh`.

#### Форк онтологии (Ontology Fork)
`ontologyFork`

Копия Project (с его парной Ontology) в пространство пользователя с созданием ссылки на исходный проект (upstream). Fork = базовый механизм для работы с демо-проектами и публичными онтологиями. При форке: создаётся новый Project с `upstream_project_id = <source_id>`, visibility = `private`, пользователь становится Owner. Fork реализуется через `POST /api/v1/projects/{id}/fork` (REST, требует `Idempotency-Key`). Связь с upstream позволяет в будущем (post-MVP) реализовать merge-upstream-changes (pull updates from source).

#### Snapshot (Release)
`snapshotRelease`

Опубликованный read-only снимок онтологии на конкретном коммите. Материализуется в отдельном read-only serving слое (CQRS: чтения ≫ записи) и экспонируется как `/projects/{pid}/releases` (GitLab-нейминг; `snapshots` → `releases`). Отличается от живого editing workspace: snapshot виден внешним потребителям, а рабочая онтология — только внутренним участникам. Модель доступа бинарная: **public** (без auth) / **restricted** (API key). Visibility снэпшота декоррелирована от visibility Project.

#### Branch Snapshot
`branchSnapshot`

Локальное REST-доступное представление состояния ветки онтологии на конкретном коммите. REST write-эндпоинты работают с branch snapshots, НЕ с живой онтологией; branch-local записи позднее коммитятся и мержатся. Является реализацией write-path invariant: прямой записи в живую онтологию не существует.

#### Последовательность построения онтологии (Ontology Build Sequence)
`ontologyBuildSequence`

Структурированное JSON-представление атомарных шагов для создания OWL-онтологии. Формируется LLM на основе содержимого загруженного документа и служит промежуточным форматом обмена между сервисом извлечения документов и сервисом онтологий. Каждый шаг содержит тип операции (создание класса, создание объектного свойства, создание свойства-литерала, установка родителя, добавление аннотации, создание индивида и др.), идентификатор сущности и опциональные атрибуты (наименование, описание, родительский класс, область определения, область значений, тип данных, характеристики). Перед показом пользователю последовательность проверяется по JSON-схеме. Выполняется атомарно через единую транзакцию Neo4j.

### Интеграция и API

#### REST API VEDO Core
`restApi`

Набор HTTP-эндпоинтов для программного доступа к функциям VEDO Core. Позволяет выполнять CRUD-операции с онтологиями, управлять версиями и ветками, получать метрики. Используется отраслевыми решениями (VEDO Family, VEDO Agro) для интеграции.

Каноническая структура — **project-scoped**: операции вложены под `/projects/{pid}/` (repository content с `?ref=`, merge_requests, releases, protected_branches) в соответствии с GitLab-выравниванием. Write-операции — **branch-scoped**: работают с branch snapshots, а не с живой онтологией. Пути `/api/v1/ontologies/*` — **deprecated** (заголовки `Deprecation`/`Sunset`; после окна миграции — `501` с `x-vedo-status: planned`). `ontology_id` — внутренний идентификатор, не экспонируется в REST-поверхности как канонический путь.

#### Webhook онтологии
`ontologyWebhook`

Механизм асинхронного уведомления внешних систем о событиях в онтологии (создание/изменение/удаление узлов, коммиты). Внешняя система подписывается на события, указывая URL-адрес для доставки уведомлений в формате JSON.

#### gRPC API
`grpcApi`

Высокопроизводительный протокол взаимодействия между внутренними микросервисами VEDO Core. Использует бинарную сериализацию Protocol Buffers и обеспечивает потоковую передачу данных (streaming).

### Мониторинг и наблюдаемость

#### OpenTelemetry (OTel)
`openTelemetry`

Стандартизированный фреймворк для сбора телеметрических данных: распределённая трассировка (traces), метрики (metrics) и логи (logs). Используется в VEDO Core для обеспечения наблюдаемости (observability) всех микросервисов.

#### Трассировка (Trace)
`trace`

Запись пути выполнения запроса через несколько сервисов VEDO Core. Состоит из набора спанов (spans), каждый из которых представляет выполнение отдельной операции (например, авторизация, запрос к Ontology Service, вызов Triple Store). Позволяет выявлять узкие места и ошибки.

#### RED Method
`redMethod`

Метод мониторинга микросервисов на основе трёх ключевых метрик: **R**ate (количество запросов в секунду), **E**rrors (количество ошибок), **D**uration (гистограмма времени ответа). Используется в VEDO Core для оценки производительности и надёжности.

#### Дашборд (Dashboard)
`dashboard`

Визуальный интерфейс (Grafana) для отображения метрик, логов и трассировки работы VEDO Core. Позволяет инженерам и администраторам отслеживать состояние системы в реальном времени.

### Фронтенд (VEDO Web UI)

#### Vue 3
`vue3`

Современный JavaScript-фреймворк для построения пользовательских интерфейсов. Используется в VEDO Core для реализации реактивного веб-приложения с поддержкой композиции компонентов и высокой производительностью рендеринга.

#### Composition API
`compositionApi`

Стиль написания компонентов во Vue 3, основанный на использовании функции `setup()` и композиции реактивных примитивов. Позволяет повторно использовать логику между компонентами, делая код более модульным и тестируемым.

#### Apollo Client
`apolloClient`

Библиотека управления состоянием для Vue 3, ориентированная на GraphQL. Используется в VEDO Core для централизованного управления данными онтологии через GraphQL-запросы, встроенный кэш и нормализацию данных. Локальное состояние интерфейса управляется через реактивные примитивы Vue (`ref`, `reactive`).

#### Vue Flow
`vueFlow`

Библиотека для визуализации и редактирования графов во Vue-приложениях. Используется в VEDO Core для отображения онтологии в виде интерактивной диаграммы связей между классами и свойствами (адаптированный порт React Flow).

#### Vite
`vite`

Инструмент для сборки фронтенд-приложений, обеспечивающий быстрый холодный старт и горячую замену модулей (HMR). Используется в VEDO Core для ускорения процесса разработки.

### Интерфейс пользователя

#### Визуализация графа (Graph Visualization)
`graphVisualization`

Интерактивное представление онтологии в виде графа, где узлы (nodes) соответствуют классам, свойствам или индивидам, а рёбра (edges) — связям между ними (`subClassOf`, `domain`, `range` и т.д.). Поддерживает масштабирование, фильтрацию и перемещение элементов перетаскиванием

#### Дерево классов (Class Tree)
`classTree`

Иерархическое представление классов онтологии в левой боковой панели. Поддерживает поиск, drag-n-drop для изменения иерархии и отображение количества дочерних узлов.

#### Панель свойств (Property Panel)
`propertyPanel`

Правая боковая панель VEDO Web UI, отображающая детальную информацию о выбранном элементе (классе, свойстве, индивиде): атрибуты, аннотации (label, comment), связи и историю изменений.

#### Режим сравнения версий (Diff Mode)
`diffMode`

Режим просмотра двух версий онтологии (коммитов) с визуальной подсветкой добавленных, изменённых и удалённых элементов. Представление доступно как в виде списка, так и в виде наложения графов.

#### CTA (Call-to-Action)
`cta`

Элемент пользовательского интерфейса (кнопка, ссылка), побуждающий посетителя к целевому действию: регистрации, знакомству с демо-проектами, началу работы с платформой. На общедоступной посадочной странице VEDO Hub используется два элемента призыва к действию: **основной** («Создать аккаунт бесплатно» — зрительно выделяющаяся кнопка, ведущая к входу через Keycloak) и **дополнительный** («Смотреть демо» — контурная кнопка, прокручивающая страницу к витрине VEDO Demos). Путь от посадочной страницы до регистрации — не более трёх нажатий, включая основной призыв.

### Инфраструктура и DevOps

#### Канонический Turtle
`canonicalTurtle`

Нормализованная форма сериализации RDF-графа в формате Turtle, при которой триплеты упорядочены лексикографически для обеспечения детерминированного представления. Используется в VEDO Core для хранения дельт коммитов, экспорта в Git-репозитории и выгрузки результатов SPARQL-запросов, когда пользователю нужен стабильный machine-readable RDF-результат.

#### LOP (Large Object Promisor)
`lop`

Экспериментальное расширение Git, предназначенное для выгрузки больших бинарных объектов (blobs) в специализированные remote-репозитории (Object Storage, S3). Перспективная альтернатива Git LFS, рассматриваемая для будущего использования в VEDO Core.

#### Helm Chart
`helmChart`

Пакет предварительно сконфигурированных Kubernetes-ресурсов для развёртывания VEDO Core. Включает деплойменты всех микросервисов, сервисы, конфигурационные карты и настройки мониторинга (Prometheus, Grafana, Tempo, Loki).

#### MinIO
`minIO`

Объектное хранилище, совместимое с S3. Используется в VEDO Core для хранения бинарных данных (изображений, файлов, эмбеддингов) при работе с индивидами и для будущей интеграции с LOP.

### Тип свойства (PropertyType)
`propertyType`

Перечисление в GraphQL-схеме VEDO Core, классифицирующее свойство онтологии:
- `OBJECT` — ссылочное свойство (ObjectProperty), связывающее индивида с другим индивидом.
- `DATATYPE` — свойство-литерал (DatatypeProperty), связывающее индивида с конкретным значением (число, строка, дата).
- `ANNOTATION` — аннотационное свойство (AnnotationProperty), используемое для добавления метаданных к классам, свойствам и индивидам (например, `rdfs:label`, `rdfs:comment`). Аннотационные свойства не участвуют в логическом выводе (reasoning).

### Аннотационное свойство (Annotation Property)
`annotationProperty`

Тип свойства в OWL, предназначенный для добавления метаданных (аннотаций) к сущностям онтологии. В отличие от объектных свойств и свойств-литералов, аннотационные свойства не учитываются reasoner'ом при логическом выводе. В GraphQL-схеме VEDO Core представлены значением `ANNOTATION` в перечислении `PropertyType`.

#### WORM (Write Once Read Many)
`worm`

Хранилище с политикой однократной записи и многократного чтения, защищающее данные от изменения или удаления. В VEDO Core используется для audit trail (7 лет), compliance evidence и immutable backup. Реализуется через S3 Object Lock в режиме COMPLIANCE с отдельными IAM-ролями, запрещающими delete/overwrite.

#### Support DB
`supportDb`

Отдельный PostgreSQL-кластер для хранения support metadata: tenant identity, контакты администраторов, SLA tier, escalation matrix, deployment config overrides. Изолирован от tenant DB (Neo4j/Version Store), Multi-AZ, синхронная репликация, PITR backup 30 дней. Доступен через `vedo-cli support` команды.

#### Support Metadata
`supportMetadata`

Данные, необходимые команде поддержки для идентификации tenant, связи с администратором, определения SLA, получения audit trail и выполнения break-glass процедуры. Включает tenant identity, контактную информацию, SLA tier, escalation contacts, audit trail критических операций, emergency access keys, compliance evidence. Хранится в изолированном хранилище (Support DB, WORM S3, Vault/KMS), не зависящем от работоспособности tenant DB.

#### Cobra
`cobra`

Фреймворк для построения CLI-приложений на Go (`spf13/cobra`). Используется в `vedo-cli` для организации иерархии команд, persistent flags, автодополнения bash/zsh/fish/powershell и генерации документации. Является де-факто стандартом для сложных многоуровневых CLI в Go-экосистеме (Kubernetes, Docker, GitHub CLI, Hugo). Решение зафиксировано в `ADR-IMPL.STACK.vedo-cli-framework-strategy`.

### Безопасность и управление доступом

#### RBAC (Role-Based Access Control)
`rbac`

Модель управления доступом на основе ролей. В VEDO Core роли `Viewer`, `Editor`, `Maintainer`, `Owner` определяют права пользователя в рамках GitLab-like модели Group/Project/Members. `Owner` управляет членством и ролями, `Maintainer` управляет workflow онтологии, включая review и merge, но не управляет членством.

**Publish gate:** публикацию (release) выполняет Maintainer+ (вес роли ≥ 2), отдельной роли publisher нет; Owner (вес = 3) также может публиковать.

**Capability scopes для нечеловеческих акторов (AI-агенты, внешние системы):** отдельная модель от role ladder — `read`, `propose:abox`, `propose:tbox`, `mr:create`, `publish:restricted`; запрещены `write:direct`, `merge:self`, `publish:public` (требует человека).

Роль `Editor` (Developer) работает в рамках branch-модели: изменения вносятся в ветку (branch snapshot) и применяются через MR — не напрямую в основную ветку.

#### Keycloak
`keycloak`

Продукт с открытым исходным кодом для управления идентификацией и доступом (IAM). Используется в VEDO Core в качестве централизованного Identity Provider (IdP) с поддержкой SSO, LDAP/Active Directory интеграции и аутентификации по протоколам OAuth2/OIDC.

#### JWT (JSON Web Token)
`jwt`

Компактный и самодостаточный способ передачи данных между сторонами в виде JSON-объекта. Используется в VEDO Core для аутентификации и авторизации клиентов (веб-интерфейса и API-клиентов).

#### OAuth2 / OIDC
`oauth2Oidc`

Протоколы делегированной авторизации и аутентификации. В VEDO Core используются через Keycloak и внешних identity providers, включая российские OAuth-провайдеры и Google.

#### Identity Provider (IdP)
`identityProvider`

Сервис, который подтверждает личность пользователя и выдаёт токены аутентификации. В VEDO Core роль IdP выполняет Keycloak или внешний OAuth/OIDC-провайдер.

#### GDPR
`gdpr`

Европейский регламент защиты персональных данных. Влияет на retention, account closure, secure erase, data residency и процедуры удаления данных.

#### 152-ФЗ / ПДн
`fz152PersonalData`

Российские требования к обработке персональных данных. Для VEDO Core важны при deployments в РФ, госсекторе, on-premise и regulated enterprise-сценариях.

#### BYOL (Bring Your Own License)
`byol`

Модель, при которой заказчик предоставляет собственную лицензию на сторонний runtime-компонент, например Neo4j Enterprise. В VEDO Core используется для отделения MIT-кода от коммерческих лицензий runtime-зависимостей.

#### Экстренный доступ (Break-glass / Emergency Admin)
`emergencyAdmin`

Механизм экстренного доступа к системе при отказе основного поставщика идентификации (Keycloak или внешний IdP). Реализуется как отдельная учётная запись вне Keycloak с паролем, хранящимся в виде стойкого хэша (bcrypt/argon2) и разделённым на части по схеме Шамира (M из N частей). Активируется через специальный endpoint `/emergency/login`, не зависящий от IdP. Возможности ограничены критическими операциями восстановления: `vedo-cli emergency readonly`, `vedo-cli restore`, `vedo-cli diagnose`, а также управлением учётными записями администраторов. Каждая активация аудируется и сопровождается уведомлением команды безопасности. Пароль ротируется каждые 90 дней и после каждого использования.

#### TOTP (Time-based One-Time Password)
`totp`

Одноразовый пароль на основе времени, генерируемый аутентификационным приложением (Google Authenticator, Authy и т.д.). Используется как второй фактор (MFA) в `vedo-cli` для интерактивного режима аутентификации: после ввода имени пользователя и пароля CLI запрашивает TOTP-код, который верифицируется через Keycloak. Допустимый второй фактор согласно `ADR-DES.SECURITY.mfa-critical-ops-mandate`. Решение зафиксировано в `ADR-DES.SECURITY.cli-mfa-strategy`.

#### ROPG (Resource Owner Password Grant)
`ropg`

OIDC grant type, при котором клиент (в данном случае `vedo-cli`) отправляет имя пользователя, пароль и TOTP-код напрямую в Keycloak для получения `access_token`. Используется в `vedo-cli` для интерактивного режима аутентификации, поскольку CLI не имеет браузера для стандартного authorization code flow. ROPG допустим только для trusted CLI-клиента (public client в Keycloak) и используется исключительно с TOTP claim — пароль без TOTP недостаточен для операций категорий A/B/C. Решение зафиксировано в `ADR-DES.SECURITY.cli-mfa-strategy`.

#### Service Account (учётная запись службы)
`serviceAccount`

Учётная запись в Keycloak, предназначенная для неинтерактивной аутентификации (CI/CD, автоматизация). Использует `client_credentials` grant для получения `access_token` без участия человека. Service account в `vedo-cli` не имеет MFA, но его scope ограничен предопределённым набором операций (категория D, read-only, backup create) — операции категорий A/B через service account запрещены на уровне Keycloak client scopes. Audit логирует `actor=service-account:<client_id>` для отличения автоматизированных операций от человеческих. Решение зафиксировано в `ADR-DES.SECURITY.cli-mfa-strategy`.

**Внешние AI-агенты** (DMZ, не часть VEDO) также используют machine identity: к service account привязываются **capability scopes** (`read`, `propose:abox`, `propose:tbox`, `mr:create`, `publish:restricted`), а записи ограничены собственным namespace `branches/{client}/*`. Никогда — `write:direct`, `merge:self`, `publish:public`.

#### M2M-токен (Machine-to-Machine token)
`m2mToken`

Токен доступа, получаемый service account'ом через `client_credentials` grant в Keycloak. Используется для автоматизированного взаимодействия между системами без участия человека. В `vedo-cli` передаётся через флаг `--token` и позволяет выполнять команды в CI/CD-пайплайнах без интерактивного ввода. Scope токена ограничен настройками клиента в Keycloak.

Для внешних AI-агентов scope токена выражается через **capability scopes** (см. `capabilityScope`): read, propose:abox, propose:tbox, mr:create, publish:restricted. Запрещены write:direct, merge:self, publish:public.

#### AuthSessionManager
`authSessionManager`

Компонент `vedo-cli`, отвечающий за управление сессией аутентификации: выполнение OIDC ROPG+TOTP login, получение и кэширование `access_token`/`refresh_token`, проверку срока действия токенов, поддержку service account токенов. `refresh_token` кэшируется в зашифрованном виде в `~/.vedo/config.yaml`. Взаимодействует с `SecurityGuard` для проверки MFA-статуса для категорий A/B. Компонент зафиксирован в C4-диаграмме `vedo-cli` и `ADR-DES.SECURITY.cli-mfa-strategy`.

#### Схема разделения секрета Шамира (Shamir Secret Sharing)
`shamirSecretSharing`

Криптографический метод разделения секрета на N частей (shares), где для восстановления секрета требуется M из N частей (пороговое значение). В VEDO Core используется для хранения пароля Emergency Admin (`ADR-DES.SECURITY.break-glass-access-strategy`): пароль разделяется между M хранителями (например, 3 из 5 senior-инженеров), что предотвращает злоупотребление одним лицом. Компенсирует отсутствие MFA на emergency path — компрометация одного хранителя недостаточна для получения полного доступа.

#### BOLA (Broken Object Level Authorization)
`bola`

Уязвимость, возникающая, когда API не проверяет, имеет ли аутентифицированный пользователь право доступа к конкретному объекту по его ID. Атакующий подменяет идентификатор объекта (tenantId, ontologyId, branchId) и получает доступ к чужому объекту. #1 угроза OWASP API Security Top 10. В VEDO Core закрывается negative authorization тестами четырёх типов (A: cross-tenant, B: cross-object, C: BFLA, D: IDOR).

#### BFLA (Broken Function Level Authorization)
`bfla`

Уязвимость, при которой пользователь с недостаточными привилегиями вызывает функцию (эндпоинт), требующую более высокой роли. Подмена роли или вызов admin-эндпоинта рядовым пользователем. В VEDO Core тестируется через negative тесты типа C.

#### IDOR (Insecure Direct Object Reference)
`idor`

Частный случай BOLA, когда объектный ID является предсказуемым (integer, base64-encoded UUID, последовательный идентификатор), что упрощает атаку перебором. В VEDO Core тестируется через negative тесты типа D (fuzzing предсказуемых ID). Ответ всегда 403, никогда 404.

#### Negative Authorization Test
`negativeAuthorizationTest`

Тест, проверяющий, что запрос без прав доступа к объекту или функции возвращает HTTP 403. Обязательный тип тестирования для всех P0 endpoint-классов VEDO Core. Четыре типа: A (cross-tenant BOLA), B (cross-object BOLA), C (BFLA), D (IDOR). Проходит в CI при каждом MR, блокирует слияние при падении любого P0 теста.

#### SAST (Static Application Security Testing)
`sast`

Инструмент статического анализа исходного кода для поиска мест, где отсутствует проверка авторизации. В VEDO Core обязателен в CI: скрипт `check-authz-coverage.py` сверяет OpenAPI-спецификацию с кодом на наличие авторизационных проверок. Блокирует MR при обнаружении endpoint без авторизации.

#### DAST (Dynamic Application Security Testing)
`dast`

Инструмент динамического анализа запущенного приложения для поиска уязвимостей. В VEDO Core рекомендуется для staging-окружения, non-blocking. Дополняет SAST и negative тесты, проверяя runtime-поведение.

#### OWASP ASVS
`owaspAsvs`

Open Web Application Security Project Application Security Verification Standard — стандарт верификации безопасности приложений. В VEDO Core применяются разделы V4 (Access Control — проверка прав доступа к объектам) и V5 (Authorization — проверка прав на выполнение функций). Negative тесты BOLA/BFLA обеспечивают выполнение требований V4.1, V4.2 и V5.1.

#### SCA (Software Composition Analysis)
`sca`

Анализ состава программного обеспечения для выявления уязвимостей в зависимостях. В VEDO Core обязателен в CI для всех языков: RustSec (Rust), Trivy (контейнеры), Safety (Python), Govulncheck (Go). Каждый MR проверяет зависимости на известные CVE.

#### SBOM (Software Bill of Materials)
`sbom`

Машиночитаемый перечень всех компонентов программного обеспечения. В VEDO Core генерируется в форматах SPDX 2.3 и CycloneDX 1.5 через `vedo-cli sca sbom generate`. Хранится вместе с релизными артефактами, доступен для аудита и supply chain верификации.

#### CVE (Common Vulnerabilities and Exposures)
`cve`

Стандартизированный идентификатор известной уязвимости в программном обеспечении. В VEDO Core SCA-сканирование проверяет зависимости на наличие известных CVE и блокирует MR при обнаружении критических (CRITICAL/HIGH) уязвимостей.

#### Advisory Database
`advisoryDatabase`

Локальная база данных известных уязвимостей для SCA-сканирования. В VEDO Core включает RustSec, Go VulnDB, PyPA Advisory DB, npm Advisory DB, NVD, GitHub Advisory DB. Для air-gapped развёртывания поставляется через `vedo-cli airgap prepare --advisory-bundle`.

#### Compliance Evidence
`complianceEvidence`

Документы и записи, подтверждающие соответствие требованиям безопасности и регуляторным нормам: SOC2 Type II report, ISO 27001 сертификат, DPA, результаты пентестов. В VEDO Core хранятся в WORM S3 с индексом в Support DB. Управляются через `vedo-cli support compliance-evidence`.

#### DDoS (Distributed Denial of Service)
`ddos`

Распределённая атака типа «отказ в обслуживании», нацеленная на исчерпание ресурсов системы (сеть, CPU, память, соединения). В VEDO Core классифицируется по уровням: L3/L4 (SYN flood, UDP amplification, ICMP flood) — защита облачным провайдером; L7 (HTTP flood, Slowloris, HTTP/2 Rapid Reset) — защита через API Gateway (WAF, rate limiting, таймауты). Подробнее: `REQ-SEC-70`.

#### WAF (Web Application Firewall)
`waf`

Межсетевой экран уровня приложений (L7), анализирующий HTTP/HTTPS трафик для обнаружения и блокировки веб-атак (OWASP Top 10, SQL-инъекции, XSS, bot-трафик). В VEDO Core WAF с правилами OWASP CRS является обязательным компонентом защиты API Gateway для production-окружений. Для SaaS — встроенный WAF облачного провайдера; для on-premise — рекомендуется Cloudflare или аналог.

#### Rate Limiting
`rateLimiting`

Ограничение частоты запросов от клиента (по IP, tenant, endpoint) для защиты от перегрузки и DDoS-атак. В VEDO Core реализовано на API Gateway с порогами: неавторизованные — 100 req/min, авторизованные — 1000 req/min, per tenant — 5000 req/min, SPARQL endpoint — 30 req/min per IP. При превышении возвращается HTTP 429.

#### API Key
`apiKey`

Учётные данные машинной аутентификации для restricted-доступа к опубликованным снэпшотам (releases). Передаётся в заголовке `X-VEDO-API-Key`; привязан к конкретному release. Генерация/отзыв через `/projects/{pid}/api_keys` (post-MVP) или project settings. Rate limiting per-API-key.

#### Capability Scope
`capabilityScope`

Модель разрешений для нечеловеческих акторов (AI-агенты, внешние системы) — НЕ role ladder. Определяет, что внешняя идентичность МОЖЕТ делать: `read`, `propose:abox`, `propose:tbox`, `mr:create`, `publish:restricted`. Никогда не включает `write:direct`, `merge:self`, `publish:public` (требует человека). AI пишет только в собственные ветки `branches/{client}/*`; proposer ≠ approver ≠ publisher; review gate = детерминированный код, не «AI ревьюит AI».

#### Write-Path Invariant
`writePathInvariant`

Архитектурное ограничение: НЕ существует пути мутации состояния онтологии вне конвейера версионирования. Каждая мутация = branch → commit → MR → review → merge. Применяется ко всем: люди, AI-агенты, администраторы. Каждая мутация — залогированный коммит с аудиторским следом (кто, что, когда, ветка). REST write-эндпоинты работают с branch snapshots. Текущий код, нарушающий инвариант, помечен для миграции в M10.

#### gitleaks
`gitleaks`

Инструмент статического анализа для обнаружения секретов (ключей API, токенов, паролей) в Git-репозиториях. В VEDO Core используется как pre-commit hook (рекомендован) и в CI (обязателен). Block-секреты (production ключи) блокируют MR; Warning (тестовые ключи) не блокируют. Настройка исключений — через `.gitleaks.toml` (утверждается Security Lead). Подробнее: `REQ-SEC-80`.

#### Secret Scanning
`secretScanning`

Процесс автоматического обнаружения секретов (токенов, ключей API, паролей, сертификатов) в исходном коде, Docker-образах и CI-логах. В VEDO Core реализован на двух уровнях: pre-commit (gitleaks) и CI (gitleaks + GitLab Secret Detection). Пороги: Block (production секреты — блокировка MR), Warning (тестовые секреты — предупреждение), Ignore (плейсхолдеры). Подробнее: `REQ-SEC-80`.

#### Secret Rotation
`secretRotation`

Процедура плановой или экстренной замены криптографических ключей и учётных данных. В VEDO Core установлены сроки ротации: JWT signing keys — 90 дней, service account tokens — 90 дней, database passwords — 180 дней, API-ключи интеграций — 180 дней. Экстренная ротация при компрометации: revoke в течение 1 часа, генерация нового секрета, деплой, уведомление клиентов. Ответственный: Security Lead. Подробнее: `REQ-SEC-80`.

---

## 8. SLA, метрики и восстановление

### RTO (Recovery Time Objective)
`rto`

Максимально допустимое время восстановления системы после отказа. Для VEDO Core: критический отказ — 4 часа, высокий — 1 час, средний — 8 часов, низкий — 24 часа.

### RPO (Recovery Point Objective)
`rpo`

Максимально допустимый объём потерянных данных при восстановлении. Определяет частоту резервного копирования. Для VEDO Core: TBox — 5 минут, ABox — 15 минут, коммиты — 1 час.

### SLA (Service Level Agreement)
`sla`

Соглашение об уровне сервиса — документированное обязательство по доступности и производительности. Для VEDO Core: MVP SaaS — доступность 99.9%, Enterprise Production — 99.95%, Premium Enterprise — 99.99% по отдельному согласованию.

### SLO (Service Level Objective)
`slo`

Целевой измеримый уровень сервиса, например availability 99.9% или latency p95 <= 120 ms. SLO задаёт планку качества, которую система должна соблюдать.

### SLI (Service Level Indicator)
`sli`

Фактическая метрика, по которой измеряется выполнение SLO: uptime, latency, error rate, probe success или доля успешных операций.

### Error Budget
`errorBudget`

Допустимый запас ошибок или простоя за период, вытекающий из выбранного SLO. Если error budget исчерпан, команда должна приоритизировать стабильность над новыми изменениями.

### P0/P1/P2/P3 Alert Severity
`alertSeverityModel`

Четырёхуровневая модель критичности алертов: P0 Critical (потеря связности, OOM, риск потери данных, критическая задержка — реакция 15 минут), P1 High (деградация сервиса, превышение SLO — реакция 1 час), P2 Warning (превышение порогов, требующее внимания — реакция 24 часа), P3 Info (информационные уведомления). P0/P1 отправляются в PagerDuty/Opsgenie и Slack/Teams; лимит усталости: P1+ ≤ 5 в день, доля ложных P0 < 1%.

### Alert Fatigue
`alertFatigue`

Снижение качества реакции на алерты из-за их избыточности: операторы перестают обрабатывать оповещения, пропускают настоящие инциденты. В VEDO Core предотвращается лимитом усталости (P1+ ≤ 5/день), долей ложных P0 < 1% и еженедельным разбором алертов для деактивации шумных сигналов.

### p95 / p99 latency
`tailLatency`

95-й и 99-й перцентили времени ответа. Используются для оценки хвостовых задержек, которые не видны по среднему времени ответа.

### Availability SLO
`availabilitySlo`

SLO доступности системы. Для VEDO Core: MVP SaaS — 99.9%, Enterprise Production — 99.95%, Premium Enterprise — 99.99% по отдельному согласованию.

### Maintenance Window
`maintenanceWindow`

Заранее объявленное окно планового обслуживания. В VEDO Core плановое обслуживание не входит в SLO только при соблюдении уведомления и лимитов длительности.

### GitOps
`gitops`

Методология управления инфраструктурой и конфигурацией через Git-репозиторий. Все изменения инфраструктуры кодируются в Terraform/Helm-файлы и проходят code review. В VEDO Core используется для деплоймента через `git push` с автоматическим применением изменений.

### CI/CD (Continuous Integration / Continuous Deployment)
`cicd`

Практика автоматической сборки, тестирования и развёртывания кода. В VEDO Core: GitHub Actions/GitLab CI для пайплайнов сборки, тестирования и деплоймента в кластер Kubernetes.

### WAL (Write-Ahead Logging)
`wal`

Механизм обеспечения целостности базы данных, при котором записи о планируемых изменениях сохраняются в лог до применения к основным данным. Используется в PostgreSQL и Neo4j для обеспечения RPO и восстановления после сбоев.

### Откат (Rollback)
`rollback`

Возврат системы, данных, конфигурации или API к предыдущему работоспособному состоянию. В VEDO Core применяется для производственных развёртываний, миграций, версий API и восстановления после ошибок.

### Защита окружения (Environment Guard)
`environmentGuard`

Механизм защиты от выполнения разрушительных команд в производственном окружении. При попытке выполнить команду уровня G2+ (см. защитные ограничения для destructive-команд) в окружении, помеченном как `production`, CLI выводит предупреждение и требует точного ввода имени окружения для подтверждения. Для SaaS дополнительно проверяется, что команда инициирована из утверждённой управляющей сети. Механизм предотвращает ошибки, при которых оператор намеревался работать со staging-окружением, но команда была применена к production-окружению.

### Эксплуатационная инструкция (Runbook)
`runbook`

Пошаговая эксплуатационная инструкция для типовой операции: миграции, отката, восстановления, вывода из эксплуатации или реагирования на инцидент.

### Incident Commander / Communications Lead / Operations Lead
`incidentCommander` / `incidentRoles`

Роли при P0/P1 инциденте: Incident Commander (назначается ≤ 5 мин для P0) — владелец процесса восстановления, принимает решения; Communications Lead (≤ 10 мин) — отвечает за коммуникации с заказчиком и командой; Operations Lead (≤ 5 мин) — отвечает за технические операции восстановления. Таймеры эскалации запускаются с момента подтверждения инцидента.

### Матрица эскалации (Escalation Matrix)
`escalationMatrix`

Формальная маршрутизация инцидентов по уровням поддержки L1–L6 с фиксированными таймерами: P0 L1→L2 за 15 мин, L2→L3 за 30 мин, L3→L4 за 1 час, L4→L5 за 2 часа, L5→L6 за 4 часа. Для P1: L1→L2 за 30 мин, L2→L3 за 90 мин, L3→L4 за 4 часа, L4→L5 за 8 часов. Таймеры запускаются с момента подтверждения инцидента, а не с ручной обработки тикета.

### Период охлаждения (Cooling-off Period)
`coolingOffPeriod`

Период (0-30 дней) между деактивацией аккаунта и необратимым удалением данных. Защищает от случайного закрытия, даёт время на передачу онтологий и восстановление доступа. В VEDO Core: аккаунт деактивирован, но данные не удалены; пользователь может восстановить доступ в течение этого периода.

### Учебная тренировка (Drill)
`drill`

Практическая проверка эксплуатационного сценария в изолированном окружении, подтверждающая, что администратор способен выполнить процедуру (восстановление, вывод из эксплуатации, миграцию) без потери данных, ошибок последовательности и обращения в поддержку. Результат тренировки фиксируется как доказательство готовности релиза. Примеры: тренировка восстановления, тренировка вывода из эксплуатации, настольная тренировка инцидента.

### PITR (Point-in-Time Recovery)
`pitr`

Восстановление данных на конкретный момент времени с использованием snapshot'ов, WAL или transaction logs. Применяется для снижения потерь при авариях и ошибочных изменениях.

### Срок хранения резервных копий (Backup Retention)
`backupRetention`

Политика сроков хранения резервных копий по типам данных и классам хранения. В VEDO Core различается для TBox, ABox, Version Store, LFS и audit logs.

### 3-2-1 Backup Strategy
`threeTwoOneBackup`

Стратегия защиты данных, требующая как минимум 3 копии данных, на 2 разных носителях, 1 из которых — вне основной площадки (off-site). В VEDO Core: local fast restore copy (быстрое восстановление), S3/MinIO primary backup (основное хранилище), off-site/cold copy (защита от потери площадки, ransomware и ошибок администратора).

### Неизменяемая резервная копия (Immutable Backup)
`immutableBackup`

Резервная копия, защищённая от удаления или перезаписи даже при компрометации production-учётных данных. Реализуется через Object Lock (WORM) на S3/MinIO или через версионирование bucket с защитой от удаления версий. Минимальные требования: отдельный bucket в другом регионе, доступ production-окружения только на запись (без права удаления), отдельное хранение учётных данных, управление bucket из отдельной административной сети. Защищает от сценария, при котором злоумышленник с доступом к production-окружению удаляет и рабочие данные, и резервные копии.

### Secure Erase
`secureErase`

Гарантированное удаление данных через физическое уничтожение носителя, перезапись или уничтожение ключей шифрования. В VEDO Core применяется в сценариях account closure, decommission и regulated enterprise.

### Audit Cold Archive
`auditColdArchive`

Долгосрочное архивное хранение audit logs в дешёвом или автономном хранилище. Нужно для расследований, compliance и корпоративных/on-premise требований к срокам хранения.

### ERP (Enterprise Resource Planning)
`erp`

Корпоративная информационная система для управления ресурсами предприятия (финансы, HR, закупки, логистика). Интеграция VEDO Core с ERP позволяет автоматически обновлять онтологию при изменениях в справочниках.

### PLM (Product Lifecycle Management)
`plm`

Система управления жизненным циклом изделия. Интеграция VEDO Core с PLM позволяет хранить онтологию продукта и связанные с ней данные о компонентах, спецификациях и процессах.

### MES (Manufacturing Execution System)
`mes`

Система управления производственными операциями. Интеграция VEDO Core с MES обеспечивает связь между онтологической моделью производства и оперативными данными цеха.

### Geo-репликация
`geoReplication`

Репликация данных между хранилищами в разных географических локациях. Используется для LFS-объектов VEDO Core (RPO: 24 часа) и обеспечения disaster recovery.

#### Переключение при отказе (Failover)
`failover`

Процесс автоматического или ручного переключения операций с отказавшего региона (А) на резервный регион (Б). В VEDO Core включает фазы: обнаружение (0-5 мин), pre-flight checks региона Б, execution (переключение БД, DNS, разогрев кэша), validation (smoke test, health check). Выполняется по failover decision matrix с блокировкой write в регионе А до завершения.

### Плавная деградация (Graceful Degradation)
`gracefulDegradation`

Режим работы при частичном отказе компонентов, когда недоступные функции блокируются, а остальные продолжают работать. В VEDO Core: при отказе Ontology Service — read-only режим, при отказе Versioning Service — недоступны коммиты, редактор остаётся рабочим. Пользователь получает не HTTP 503, а ограниченный, но работоспособный интерфейс.

### Предохранитель (Circuit Breaker)
`circuitBreaker`

Шаблон отказоустойчивости: при превышении порога ошибок (failure_threshold) вызов сервиса блокируется на заданный timeout, после чего допускается пробный запрос (half-open). В VEDO Core применяется для Versioning Service (3 ошибки, 60s timeout) и Ontology Service (3 ошибки, 30s timeout) для предотвращения каскадных отказов.

#### Предварительная проверка (Pre-flight Check)
`preflightCheck`

Набор проверок, выполняемых перед началом failover: доступность региона Б, статус репликации Neo4j/PostgreSQL, ёмкость storage, health API Gateway/сервисов, актуальность backup, ready-статус Redis/RabbitMQ. В VEDO Core все 6 pre-flight checks должны пройти успешно, иначе failover прерывается с эскалацией.

### NPS (Net Promoter Score)
`nps`

Метрика лояльности пользователей: «Какова вероятность, что вы порекомендуете VEDO Core коллеге?» (0–10). Классификация: 9–10 — промоутеры, 7–8 — нейтралы, 0–6 — критики. NPS = % промоутеров − % критиков. Целевое значение: ≥ 40. Измеряется ежеквартально через in-app опрос. Источник: `usability-metrics.md`.

### CES (Customer Effort Score)
`ces`

Метрика усилия пользователя: «Насколько легко было выполнить задачу?» (1–5). Целевое значение: ≥ 4. Измеряется после каждого обращения в поддержку или завершения ключевого сценария через in-app опрос. Источник: `usability-metrics.md`.

### UX-долг (UX Debt)
`uxDebt`

Принятые в релиз отклонения от acceptance criteria или визуального эталона, которые не были исправлены до мержа. Классифицируется по категориям P0–P3: P0 (критический, блокирует сценарий) — исправление не более 1 релизного цикла, P1 (значимый) — не более 2 циклов, P2 (косметический) — не более 4 циклов, P3 (улучшение) — без ограничений (backlog). P0 и P1 являются release blocker. Источник: `ux-review-process.md`.

---

## 9. Модели развёртывания

### SaaS (Software as a Service)
`saas`

Модель развёртывания, при которой приложение размещается в облаке и предоставляется как сервис по подписке. Для VEDO Core это мультитенантная SaaS-модель на старте (P0): VEDO управляет сервисом, а клиенты получают доступ через браузер.

### On-premise
`onPremise`

Модель развёртывания, при которой ПО устанавливается и управляется на серверах заказчика. Для VEDO Core это Enterprise P2: заказчик управляет обновлениями, резервными копиями, SSL и мониторингом. Требуется для финтеха, госсектора и сценариев с требованиями 152-ФЗ.

### Приватное облако (Private Cloud)
`privateCloud`

Модель развёртывания, при которой облачные ресурсы выделены только одному клиенту (физическая изоляция). VEDO управляет ПО, заказчик предоставляет инфраструктуру (IaaS). Для VEDO Core это P3-сценарий по запросу корпоративных клиентов.

### Мультитенантность (Multi-tenant)
`multiTenant`

Архитектура, при которой один экземпляр приложения обслуживает нескольких клиентов (тенантов) с логической изоляцией данных. В SaaS-версии VEDO Core изоляция обеспечивается через namespace, `ontology_id` и отдельный realm Keycloak на каждого тенанта.

### IaaS (Infrastructure as a Service)
`iaas`

Модель облачных вычислений, предоставляющая виртуальные машины, хранилище и сеть по запросу. Используется в VEDO Core для приватного облака (клиент предоставляет VMs/k8s, VEDO разворачивает приложение).

### BYOC (Bring Your Own Cloud)
`byoc`

Модель, при которой клиент запускает SaaS-решение в собственном облаке (AWS, Azure, Yandex Cloud), но лицензия управляется централизованно. Для VEDO Core это один из вариантов приватного облака на P3.

### Kubernetes / K8s
`kubernetes`

Платформа оркестрации контейнеров. В VEDO Core является целевой средой производственного развёртывания через Helm charts.

### Docker
`docker`

Формат контейнеризации сервисов и инструменты для сборки образов. В VEDO Core используется для упаковки микросервисов и локального запуска.

### Изолированное развёртывание (Air-gapped Deployment)
`airGappedDeployment`

Развёртывание в изолированной среде без доступа к публичному интернету. Требует локального реестра образов, документации для автономного использования, локальных Helm charts и отключения внешних проверок телеметрии и версий.

### Постепенное обновление (Rolling Update)
`rollingUpdate`

Стратегия обновления, при которой старые экземпляры сервиса постепенно заменяются новыми без полной остановки системы. Используется как подход по умолчанию для minor- и patch-обновлений.

### Сине-зелёное развёртывание (Blue-Green Deployment)
`blueGreenDeployment`

Стратегия с двумя средами: текущей (blue) и новой (green). После проверки новой версии производственный трафик переключается на green; откат выполняется обратным переключением.

### Канареечный выпуск (Canary Release)
`canaryRelease`

Постепенный выпуск новой версии на небольшую долю трафика перед полным переключением. В VEDO Core является опциональным и требует service mesh.

### HA (High Availability)
`ha`

Архитектурное свойство высокой доступности за счёт репликации, автоматического переключения при отказе и устранения единых точек отказа. Требуется для Enterprise SLO 99.95% и выше.

### Multi-AZ
`multiAz`

Размещение компонентов в нескольких зонах доступности внутри одного региона. Используется для повышения отказоустойчивости без полноценного много регионального развёртывания.

### Object Storage / S3-compatible Storage
`objectStorage`

Объектное хранилище для файлов, резервных копий, LFS-объектов и архивов. S3-compatible означает совместимость с API Amazon S3, включая MinIO и другие реализации.

### Git LFS (Git Large File Storage)
`gitLfs`

Расширение Git для хранения крупных файлов вне обычной истории репозитория. Рассматривается для больших ABox-дампов, бинарных объектов и RDF-файлов, которые слишком велики для обычного Git.

### Local Registry
`localRegistry`

Локальное хранилище контейнерных образов и Helm-артефактов внутри периметра заказчика. Используется для установки в изолированном контуре через Harbor, Nexus или аналог.

### Helm Chart
`helmChart`

Пакет предварительно сконфигурированных Kubernetes-ресурсов для развёртывания VEDO Core. Включает деплойменты всех микросервисов, сервисы, конфигурационные карты и настройки мониторинга.

### Feature Flags (Feature Toggles)
`featureFlags`

Механизм включения/отключения функциональности без изменения кода или переразвёртывания. В VEDO Core используется для canary-релизов, shadow mode, поэтапного включения новых возможностей и быстрого отключения проблемной функциональности.

### Теневое развёртывание (Shadow Mode)
`shadowMode`

Режим, при котором новая версия сервиса обрабатывает запросы параллельно с текущей версией, но результаты не возвращаются клиенту. Используется для проверки корректности работы новой версии на production-трафике без влияния на пользователей. В VEDO Core применяется при мажорных миграциях и тестировании обратной совместимости API.

### CapEx (Capital Expenditure)
`capex`

Капитальные расходы на приобретение или модернизацию основных средств. В контексте VEDO Core — затраты заказчика на оборудование для on-premise-развёртывания (серверы, хранилища).

### OpEx (Operational Expenditure)
`opex`

Операционные расходы на текущее обслуживание. В контексте VEDO Core — подписка за SaaS (ежемесячная/ежегодная оплата за использование).

### ФСТЭК
`fstek`

Федеральная служба по техническому и экспортному контролю. Выдаёт сертификаты и лицензии на соответствие требованиям информационной безопасности. Для VEDO Core on-premise — требуется аттестация для госсектора.

### WCAG (Web Content Accessibility Guidelines)
`wcag`

Международные рекомендации по обеспечению доступности веб-контента. Уровни: A (минимальный), AA (стандартный), AAA (максимальный). Для VEDO Core: Level A для MVP, Level AA для P1. Регуляторные требования: ФЗ № 419-ФЗ, ГОСТ Р 52872-2019.

### ARIA (Accessible Rich Internet Applications)
`aria`

Набор атрибутов для улучшения доступности динамического веб-контента. Включает role, state, properties. Используется в VEDO Core для кастомных компонентов: combobox, modal, autocomplete.

### Screen Reader
`screenReader`

Программа чтения экрана для незрячих пользователей (NVDA, JAWS, VoiceOver). VEDO Core должен быть совместим с основными screen reader'ами на уровне WCAG 2.1 AA.

### Рефлоу (Reflow)
`reflow`

Способность контента адаптироваться к размеру окна без горизонтальной прокрутки. WCAG 2.1 AA требует работоспособности при ширине 320px. Для VEDO Core — табличный режим графа при narrow viewport.

---

## 10. Методология, архитектура и качество

### ADR Naming Convention
`adrNamingConvention`

Стандарт идентификации архитектурных решений VEDO Core в формате `ADR-<LEVEL>.<AREA>.<semantic-tag>`, где LEVEL = BIZ | DES | IMPL, AREA = API | DATA | INFRA | SECURITY | UI | PROCESS | INTEGRATION | STACK | OPS | DOC, semantic-tag — короткая англоязычная метка (kebab-case, 2-5 слов). Паттерны semantic-tag: `-vs-` (выбор альтернатив), `-vs-...-vs-` (множественный выбор), `-or-` (равнозначные варианты), `-tradeoff` (компромисс), `-adoption` (внедрение), `-mandate` (вынужденное решение), `-strategy` / `-approach` / `-pattern` (подход), `-evolution` / `-migration` (изменение существующего), `-scope` / `-boundary` (границы). Запрещены пробелы, максимальная длина semantic-tag — 40 символов.

### ADR (Architecture Decision Record)
`adr`

Запись архитектурного решения с контекстом, выбранным вариантом, альтернативами и последствиями. В VEDO Core используется для фиксации проектных решений и trade-offs.

### C4 Model
`c4Model`

Модель описания архитектуры через уровни Context, Container, Component и Code/Class. Используется для структурного описания системы и границ сервисов.

### NFR (Non-Functional Requirement)
`nfr`

Нефункциональное требование к качеству системы: производительности, безопасности, доступности, наблюдаемости, сопровождаемости или удобству использования.

### PBT (Property-Based Testing)
`pbt`

Тестирование свойств системы на множестве сгенерированных входных данных. В HLV используется для проверки инвариантов и edge cases с большим числом генераций.

### Gate
`gate`

Автоматическая или ручная проверка качества, блокирующая продвижение артефакта или релиза при невыполнении условия. Примеры: API Coverage Gate, false-conflict-test, performance gate.

### Фаззинг-тестирование (Fuzz Testing)
`fuzzTesting`

Метод тестирования, при котором система получает большое количество автоматически сгенерированных или мутированных входных данных для выявления сбоев, аварийных завершений, зависаний и уязвимостей в обработке входа. В VEDO Core применяется к парсерам и обработчикам запросов как отдельный обязательный контур качества.

### Базовая линия производительности (Performance Baseline)
`performanceBaseline`

Зафиксированный эталон метрик производительности (время ЦП, задержки p95/p99, потребление памяти), с которым сравниваются результаты тестов в CI. Используется для обнаружения регрессий и блокировки релизов при превышении согласованных порогов.

### Защита во время выполнения (Runtime Protection)
`runtimeProtection`

Набор ограничителей и проверок, действующих в работающем сервисе: тайм-ауты, лимиты размеров входа, лимиты глубины и сложности запросов, контроль памяти и рекурсии. Цель — локализовать ущерб от вредоносного или патологического ввода без остановки всего сервиса.

### Усечение результата (Result Truncation)
`resultTruncation`

Преднамеренное ограничение объёма возвращаемых данных до безопасного порога с явным уведомлением клиента (например, через предупреждение и код состояния). Применяется для предотвращения перегрузки памяти и канала при больших выборках.

### Traceability
`traceability`

Связь требования с контрактом, тестом, сценарием и gate. В HLV traceability показывает, чем именно доказано выполнение каждого требования.

### HLV (Human-Led Validation)
`hlv`

Методология, в которой human artifacts являются источником требований, contracts формализуют требования, а код и тесты являются производными артефактами.

### Saga Pattern
`sagaPattern`

Архитектурный паттерн для управления распределёнными транзакциями в микросервисной архитектуре: каждая операция публикует событие или вызывает следующий шаг; при сбое выполняются компенсирующие действия (rollback предыдущих шагов). В VEDO Core применяется для跨-сервисных операций (например, импорт онтологии с созданием коммита, обновлением кэша и уведомлением через WebSocket).

### Пробный прогон (Dry-run)
`dryRun`

Режим выполнения операции без фактического изменения данных, с возвратом плана и предварительного отчёта. В VEDO Core обязателен для опасных действий (удаление класса, массовый импорт) и операций с необратимыми последствиями. Позволяет пользователю увидеть impact preview до подтверждения.

### Чтение после записи (Read-after-write)
`readAfterWrite`

Паттерн верификации сохранения: после подтверждения записи клиент немедленно читает сохранённые данные и сверяет с ожидаемыми. В VEDO Core исключает ложный success при рассинхронизации кэша и серверной части, гарантируя, что save-состояние является проверяемым.

### Углеродный след (Carbon Footprint)
`carbonFootprint`

Количество выбросов CO₂, связанных с работой системы. В VEDO Core целевые метрики: Energy per Request < 0.5 Дж, Carbon per User < 100 г CO₂e/мес. Измеряется через Intel RAPL, PowerAPI или Kepler для Kubernetes. Отчётность опциональна для MVP и включается для корпоративных клиентов по запросу.
