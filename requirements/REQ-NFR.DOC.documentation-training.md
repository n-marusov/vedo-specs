# Документация и Обучение

| Атрибут | Значение |
|---------|----------|
| **ID** | REQ-NFR.DOC.documentation-training |
| **Уровень** | NFR |
| **Атрибут качества** | Supportability |
| **Приоритет** | P1 |
| **Статус** | УТВЕРЖДЕНО |
| **Источник** | — |
| **Критерии приёмки** | — |

---

## Назначение

Фиксирует минимально достаточные требования к обучению и документации VEDO Core для MVP, on-premise и enterprise поставок.

Документация является частью продукта: она должна позволять пользователю, администратору и разработчику выполнить базовые сценарии без обращения в поддержку и в пределах целевых usability metrics.

## Обязательные документы MVP

| Документ | Целевая аудитория | Формат | Критерий приемки |
|----------|-------------------|--------|---------------------|
| Краткое руководство пользователя | Инженеры знаний, аналитики | Markdown / HTML | Новый пользователь создаёт первый класс и выполняет навигацию по графу без внешней помощи в пределах `time-to-first-task`: <= 2 минут для новичков и <= 30 секунд для опытных пользователей по документационному сценарию |
| Административное руководство | DevOps, администраторы онтологий | Markdown / HTML | Администратор разворачивает или обновляет кластер без обращения в поддержку, следуя пошаговой инструкции |
| Руководство разработчика / API Guide | Разработчики, интеграторы | OpenAPI / Swagger + Markdown | Разработчик создаёт класс и выполняет SPARQL-запрос через API без чтения исходного кода |
| Руководство по air-gap deployment | DevOps, специалисты безопасности | Markdown / PDF или offline HTML | Администратор разворачивает VEDO Core в закрытом контуре без интернета, используя поставленные инструкции и файлы |
| Руководство по backup & restore | DevOps, инженеры поддержки | Markdown / HTML | Администратор восстанавливает tenant или ontology из backup в пределах RTO 2-4 часа |
| Руководство по диагностике инцидентов | Инженеры поддержки | Markdown / HTML | Инженер локализует причину P0-инцидента по данным OpenTelemetry за <= 10 минут |

## Критерии Приемки Для Всей Документации

### Usability

| Критерий | Способ проверки | Целевое значение |
|----------|-----------------|------------------|
| Новый пользователь выполняет базовый сценарий без внешней помощи | Usability test, минимум 5 пользователей | 100% успешных завершений |
| Время выполнения базового сценария по документации | Usability test + timing | В пределах `time-to-first-task`: <= 2 минут для новичков, <= 30 секунд для опытных в документационном сценарии |
| Документация снижает обращения в поддержку | Анализ support tickets первые 3 месяца | < 5% tickets имеют ответ, уже описанный в документации |

Базовые сценарии:

- Пользователь: создать класс, добавить свойство, создать индивида, выполнить SPARQL query через GUI.
- Администратор: развернуть кластер, настроить backups, восстановить из backup.
- Разработчик: создать класс через API, получить список классов, выполнить SPARQL query через API.

### Структура И Полнота

| Критерий | Проверка |
|----------|----------|
| Наличие всех обязательных разделов | Checklist на каждый документ |
| Оглавление с кликабельными ссылками | Ручная проверка или Antora build validation |
| Поиск по HTML-документации | Ключевые термины находятся за <= 3 clicks |
| Версионирование | На странице документа указаны версия документа и версия ПО |

### Язык И Стиль

| Критерий | Описание |
|----------|----------|
| Русский язык | Human artifacts, customer-facing docs, runbooks и guides пишутся на русском языке согласно `constraints.localization.global` |
| Инструктивный стиль | Шаги формулируются в настоящем времени и повелительном наклонении: "Нажмите", "Выберите", "Укажите" |
| Терминологическое единообразие | Термины из `glossary.md` используются одинаково во всех документах |
| Отсутствие двусмысленностей | Каждый шаг описывает одно проверяемое действие |
| Copy-paste ready commands | Команды, `curl` examples и shell snippets должны выполняться без правки, кроме явно отмеченных placeholders |

### Доступность Документации

| Канал | Требование |
|-------|------------|
| Web documentation | Доступна всем SaaS users через documentation site |
| Offline documentation | Поставляется с on-premise и air-gapped версией как PDF, архив или offline HTML bundle |
| UI hints | Сложные термины и поля имеют tooltip со ссылкой на соответствующий раздел документации |

## Docs-As-Code

- Документация хранится в Git repository вместе с кодом и human artifacts.
- Основной формат product documentation: AsciiDoc + Antora для многостраничной сборки; Markdown artifacts используются как source для HLV.
- CI/CD собирает документацию при каждом push в `main`.
- Pull request, который меняет функциональность, API, CLI, deployment или user workflow, должен обновлять документацию в том же PR/MR.
- API documentation генерируется из OpenAPI, GraphQL schema и SPARQL/API guides.
- Обновление документации после изменения кода должно попадать в тот же PR/MR; отдельное обновление допускается только как emergency fix не позднее 1 рабочего дня.

## Проверки Документации

| Проверка | Описание | Инструмент |
|----------|----------|------------|
| Spell/style lint | Проверяет орфографию и стиль вне code blocks | `typos`, `vale` |
| Link checker | Проверяет внутренние и внешние ссылки | `linkchecker`, `htmltest` |
| Executable commands | Проверяет shell/curl snippets на copy-paste готовность | `mdx`, shell smoke scripts или docs CI job |
| Antora build | Проверяет сборку site, navigation и cross references | `antora generate llm/src/docs/playbook-*.yml` |
| Usability test | Проверяет выполнение базовых сценариев по документации | Наблюдатель, видеозапись, timing |

## Роли И Ответственность

| Роль | Ответственность |
|------|----------------|
| Технический писатель | Структура, редактура, единый стиль, сборка documentation site |
| Разработчик | API, CLI, configuration и технические разделы; обновление при изменении кода |
| DevOps-инженер | Deployment, backups, monitoring, air-gap и incident runbooks |
| QA-инженер | Проверка выполнимости примеров, link checks, docs usability scenarios |

## Рекомендуемый Объём MVP

| Документ | Ориентировочный объём | Время подготовки | Приоритет |
|----------|----------------------|------------------|-----------|
| Краткое руководство пользователя | 5-7 страниц | 8 часов | P0 |
| Административное руководство | 15-20 страниц | 24 часа | P0 |
| API Guide | Swagger/OpenAPI + 5 страниц | Автоматически + 4 часа | P0 |
| Air-gap Deployment Guide | 5-10 страниц | 8 часов | P1 |
| Backup & Restore Guide | 5-7 страниц | 4 часа | P0 |
| Incident Diagnosis Guide | 10-15 страниц | 8 часов | P1 |

## Формулировка Для Заказчика

VEDO Core включает пакет технической документации: краткое руководство пользователя, административное руководство, руководство разработчика по REST/GraphQL/SPARQL/CLI, руководство по air-gap deployment, инструкции по backup/restore и руководство по диагностике инцидентов.

Документация тестируется usability-методом: новый пользователь выполняет базовые сценарии без обращения в поддержку в пределах целевого времени. Документация поставляется как HTML для web и offline bundle/PDF для on-premise и air-gapped environments.

## Бизнес-Правила

- MVP documentation package must include user onboarding, admin guide, API guide, air-gap deployment guide, backup/restore guide, and incident diagnosis guide.
- Human-readable documentation must be in Russian by default according to `constraints.localization.global`.
- Documentation must be versioned with the product release.
- Documentation must be available online for SaaS and offline for on-premise/air-gapped deployments.
- User onboarding documentation must let a new user complete the first class creation/navigation scenario within the documentation usability target.
- Admin documentation must let an administrator deploy/update the cluster and perform backup/restore without support.
- Developer documentation must let an integrator create a class and run a SPARQL query through API without reading source code.
- Documentation CI must include Antora build, link checking, spell/style lint, and command snippet validation where safe.
- Functional changes must update documentation in the same PR/MR unless an emergency exception is explicitly recorded.

## Открытые Вопросы

- Нет открытых вопросов по требованиям к обучению и документации.
