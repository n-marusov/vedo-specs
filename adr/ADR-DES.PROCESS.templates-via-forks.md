# ADR-DES.PROCESS.templates-via-forks — Замена Templates BC на Demos + Forks

**Дата:** 2026-07-22
**Статус:** Принято

## Контекст

В исходной архитектуре VEDO Core существовала концепция «Шаблон онтологии» (Ontology Template) — валидная OWL-онтология, используемая как отправная точка при создании новой онтологии. Для поддержки шаблонов планировался отдельный Bounded Context (Templates BC) с:
- Каталогом шаблонов (catalog, search, filter)
- Semantic versioning (MAJOR/MINOR/PATCH)
- Метаданными (metadata.json с 13 полями: name, description, version, author, tags, category, usage_count и др.)
- Жизненным циклом (review каждые 6 месяцев, deprecated при < 5 использований, archive)
- Ownership model (VEDO team / community / personal)

В REQ-файлах были черновики (status=ЧЕРНОВИК), в API Gateway — частичная реализация (`template_handler.go`, `ai_orch_proxy.go` — `HandleListTemplates`, `HandleApplyTemplate`), но:
- JSON-файлы шаблонов в `internal/templates/ontologies/` не были заполнены (0 загружаемых шаблонов, handler возвращает пустой список)
- GUI template flow не был реализован (только каркас во Vue SFC)
- Frontend template queries отсутствовали (grep дал 0 бизнес-совпадений)

Одновременно Social Hub (F13) — приоритет #1 в vision.md — предполагает механизм форков онтологий (F13.1). Fork = копия Project + Ontology со ссылкой на upstream. Это тот же механизм, что нужен для «применения шаблона», но без отдельного semver/catalog/lifecycle.

## Требование-источник

- `specs/vision.md` §2.1 — F13 Social Hub (приоритет #1), F14 LLM-генерация и шаблоны
- `specs/glossary.md` — «Шаблон онтологии» (Ontology Template)
- `.ai-factory/ROADMAP.md` — M2 «Template Baseline», M15 «fork/versioning APIs»
- `.ai-factory/RESEARCH.md` — DDD Context Map, сессия 2026-07-22, Deep Dive 1 (Templates)

## Решение

Заменить концепцию Templates BC на **Demos + Forks**:

1. **Templates BC выпиливается.** Шаблоны не имеют отдельного BC, catalog, semver, lifecycle, metadata.json.
2. **5 демо-проектов** в группе `VEDO Demos` (Organization, Product, Process, Glossary, Event) создаются как seed data при деплое через `deploy/seeds/vedo-demos/bootstrap.sh`.
3. **Fork** (`POST /api/v1/projects/{id}/fork`) — базовый механизм для работы с демо-проектами и публичными онтологиями. Fork = copy Project + Ontology + upstream link. Fork создаёт новый private project в пространстве пользователя; пользователь становится Owner.
4. **F13.1 Forks переносится из post-MVP в MVP.** Fork механизм един для демо-проектов и социального хаба.
5. **Git tags** заменяют semver для версионирования демо-проектов. **forks_count** заменяет usage_count. **visibility=public** + Group «My Templates» заменяют personal catalog.
6. Community PR для новых демо-проектов — через обычный MR flow (fork + merge request в `VEDO Demos`).

### Что теряется (acceptable)

| Потеря | Компенсация |
|--------|------------|
| Semantic versioning (MAJOR/MINOR/PATCH) | Git tags и commit history — богаче и проще |
| Auto-deprecate по usage_count (< 5 за 6 мес) | Ручное maintenance — для 5-20 демо-проектов достаточно |
| Personal catalog (личные шаблоны пользователя) | Group «My Templates» или фильтр owner_id=current_user + visibility=public |
| metadata.json с 13 полями | Большинство полей уже есть в Project (name, description, author=owner); tags/category — extension на Project |
| Review lifecycle (6 мес) | Устаревание проектов — обычный lifecycle Project |

### Что приобретается

| Выигрыш | Значение |
|---------|----------|
| -1 Bounded Context | 12 → 11 BC в DDD-карте |
| -1 cross-BC Saga | Нет Saga «ApplyTemplate» (Templates → Organization → Ontology) |
| -1 Published Language | Нет интерфейса «OWL document + metadata.json» |
| F13.1 в MVP | Forks — базовый механизм для templates и social hub |
| Единая механика fork | Fork для всего: демо-проекты, чужие публичные онтологии, community PR |
| Community PR через MR flow | Вместо отдельного PR-репозитория для шаблонов |
| `upstream_project_id` — MVP | Social Hub получает инфраструктуру сейчас |

## Рассмотренные альтернативы

| Альтернатива | Причина отклонения |
|--------------|--------------------|
| **A: Templates BC с catalog UX** | Преждевременная автоматизация для 5 шаблонов; lifecycle, semver, metadata — слишком тяжеловесно |
| **B: Templates как extension of Ontology BC** | Lifecycle (review/deprecate/auto-archive) — другой domain logic; загрязнение ядра онтологии |
| **C: Личный каталог шаблонов** | Заменяется group «My Templates» через стандартный механизм проектов |

## Последствия

**Положительные:**
- Упрощение архитектуры: -1 BC, -1 Saga, -1 Published Language
- Единый механизм fork для демо-проектов и социального хаба
- F13.1 переносится в MVP — закрывается vision→specs gap
- Community PR через стандартный MR flow, не через отдельный репозиторий

**Отрицательные:**
- Нет автоматического deprecate по usage (ручное maintenance VEDO team)
- Нет semver для «шаблонов» (заменяется git tags + commit history)
- Fork-зависимость от функциональности Versioning BC (copy branch)

**Меры снижения рисков:**
- Fork реализуется через Saga с compensating action (откат при ошибке Versioning)
- Git tags — более гибкий механизм версионирования, чем принудительный semver

## Связанные ADR

- [ADR-DES.SECURITY.gitlab-like-organization-model.md](ADR-DES.SECURITY.gitlab-like-organization-model.md) — fork = Project copy, upstream_project_id
- [ADR-DES.API.organization-rest-endpoints.md](ADR-DES.API.organization-rest-endpoints.md) — fork endpoint extension (`POST /api/v1/projects/{id}/fork`)
- [ADR-DES.API.rest-graphql-mutation-boundary.md](ADR-DES.API.rest-graphql-mutation-boundary.md) — fork = write, REST only, не GraphQL
