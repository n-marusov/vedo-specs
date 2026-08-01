# Прецеденты использования — VEDO Core

В этой директории находятся отдельные файлы прецедентов использования (Use Cases) проекта VEDO Core. Каждый файл соответствует одному UC и назван по шаблону `<UC-ID>.md`.

---

## Формат идентификатора UC

```
UC-<L1>.<L2>.<L3>
```

Где:

| Часть | Описание | Допустимые значения |
|-------|----------|---------------------|
| **L1** | Домен | `browse` · `editor` · `abox` · `git` · `team` · `api` · `metrics` · `io` · `admin` · `diagnostics` · `ops` · `support` · `platform` |
| **L2** | Поддомен | `classes` · `properties` · `individuals` · `tree` · `graph` · `search` · `public` · `commits` · `branches` · `merges` · `reviews` · `comments` · `integration` · `analytics` · `import` · `export` · `publish` · `system` · `access` · `backup` · `restore` · `migration` · `airgap` · `diagnose` · `trace` · `auth` · `webhooks` · `docs` · `tickets` · `knowledge` · `feedback` · `telemetry` · `roadmap` · `community` · `status` · `reports` · `forms` |
| **L3** | Семантический англоязычный тег в kebab-case, отражающий суть UC | Свободная семантическая метка (не ограничена глаголами) |

### Правила именования

1. Только **kebab-case**, без цифр и индексов.
2. **L1** — строго из списка доменов.
3. **L3** — свободная семантическая метка, отражающая конкретный сценарий.
4. UC моделируют не только GUI-сценарии: допустимы GUI, API, CLI, Webhook/Event, Schedule/Job, Support Portal.
5. Для каждого UC рекомендуется указывать метаданные `**Канал:** GUI | API | CLI | Webhook | Schedule | Portal | Mixed`.

### Примеры

| ✅ Корректно | ❌ Некорректно |
|---|---|
| `UC-editor.classes.create-new-class` | `UC-1.editor.classes` |
| `UC-git.commits.commit-changes-with-message` | `UC-editor.classes.create` (слишком коротко) |
| `UC-team.reviews.review-merge-request` | `UC-editor.classes.createClass` (camelCase) |
| `UC-support.tickets.create-and-track-support-ticket` | `UC-editor.classes.creating` (не kebab-case) |
| `UC-io.export.export-ontology-to-turtle-format` | `UC-editor.classes.1` (содержит цифру) |

---

## Формат описания UC

Каждый файл UC содержит следующие секции:

| Секция | Обязательность | Описание |
|--------|---------------|----------|
| `### <UC-ID>: <Название>` | Обязательно | Заголовок с идентификатором и кратким названием на русском |
| `**Актор:**` / `**Акторы:**` | Обязательно | Роль(и) пользователя, инициирующего прецедент |
| `**Приоритет:**` | Обязательно | `P0` (критичный), `P1` (высокий), `P2` (средний) |
| `**Ключевая функция:**` | Обязательно | Ссылка на функции из `vision.md` |
| `**Канал:**` | Рекомендуется | `GUI` / `API` / `CLI` / `Webhook` / `Schedule` / `Portal` / `Mixed` |
| `**Описание:**` | Обязательно | Краткое описание прецедента |
| `**Основной поток:**` | Обязательно | Нумерованный список шагов основного сценария |
| `**Альтернативные потоки:**` | Рекомендуется | Варианты (A1, A2, ...) с описанием отклонений |
| `**Постусловия:**` | Обязательно | Состояние системы после успешного выполнения UC |
| `**Источник требований:**` | Рекомендуется | Ссылка на файл(ы) требований |

---

## Состав директории

Директория содержит **48 файлов UC** (27 базовых + 21 запланированный).

### Базовые UC

| # | UC-ID |
|---|-------|
| 1 | `UC-editor.classes.manage-class-lifecycle` |
| 2 | `UC-editor.properties.manage-property-lifecycle` |
| 3 | `UC-editor.properties.manage-ontology-annotations` |
| 4 | `UC-abox.individuals.manage-individual-lifecycle` |
| 5 | `UC-browse.tree.view-ontology-tree-and-graph` |
| 6 | `UC-browse.graph.view-ontology-graph-with-pagination` |
| 7 | `UC-browse.search.search-ontology-elements` |
| 8 | `UC-browse.search.execute-sparql-query-through-gui` |
| 9 | `UC-metrics.analytics.view-ontology-metrics` |
| 10 | `UC-git.commits.manage-commit-history` |
| 11 | `UC-git.branches.manage-branch-workflow` |
| 12 | `UC-git.commits.compare-ontology-versions` |
| 13 | `UC-team.reviews.review-merge-request` |
| 14 | `UC-team.comments.discuss-merge-request-changes` |
| 15 | `UC-editor.classes.validate-ontology-with-shacl` |
| 16 | `UC-editor.classes.manage-data-quality-rules` |
| 17 | `UC-io.import.import-and-export-ontology-data` |
| 18 | `UC-projects.fork` |
| 19 | `UC-api.integration.integrate-through-platform-apis` |
| 19 | `UC-admin.system.manage-platform-configuration` |
| 20 | `UC-admin.access.manage-membership-and-permissions` |
| 21 | `UC-admin.backup.manage-backup-and-restore-via-cli` |
| 22 | `UC-admin.migration.apply-migrations-with-rollback-via-cli` |
| 23 | `UC-admin.airgap.prepare-air-gapped-package-via-cli` |
| 24 | `UC-diagnostics.trace.diagnose-incident-by-trace-id` |
| 25 | `UC-admin.migration.transfer-and-diff-ontologies-via-cli` |
| 26 | `UC-io.publish.publish-ontology-snapshot` |
| 27 | `UC-browse.public.view-published-ontology` |

### Запланированные UC

| # | UC-ID |
|---|-------|
| 28 | `UC-abox.individuals.export-individuals-to-xlsx` |
| 29 | `UC-abox.individuals.navigate-linked-individuals` |
| 30 | `UC-team.comments.view-project-comment-feed` |
| 31 | `UC-team.comments.enforce-comment-visibility-by-access` |
| 32 | `UC-api.auth.issue-and-validate-jwt-tokens` |
| 33 | `UC-api.webhooks.manage-subscriptions-and-delivery` |
| 34 | `UC-api.docs.view-openapi-specification` |
| 35 | `UC-metrics.analytics.view-ontology-complexity-trends` |
| 36 | `UC-io.import.import-and-export-ontology-xlsx` |
| 37 | `UC-io.export.export-ontology-to-docx` |
| 38 | `UC-io.forms.configure-import-form-templates` |
| 39 | `UC-io.reports.generate-ontology-reports` |
| 40 | `UC-support.tickets.create-and-track-support-ticket` |
| 41 | `UC-support.tickets.manage-ticket-via-vedo-cli` |
| 42 | `UC-support.knowledge.search-knowledge-base-and-faq` |
| 43 | `UC-support.feedback.submit-in-app-feedback-and-nps` |
| 44 | `UC-support.telemetry.manage-usage-telemetry-opt-in` |
| 45 | `UC-support.roadmap.vote-and-comment-roadmap-items` |
| 46 | `UC-support.community.participate-in-community-forum` |
| 47 | `UC-support.status.view-public-status-page` |
| 48 | `UC-support.feedback.review-feedback-analytics` |
| 49 | `UC-platform.landing.first-visit-and-signup` |

---

## Связанные артефакты

- [Видение продукта](../vision.md)
- [Требования](../requirements/)
- [Глоссарий](../glossary.md)
