# GUI Implementation

| Атрибут | Значение |
|---------|----------|
| **ID** | REQ-USR.UI.gui-implementation |
| **Уровень** | USR |
| **Атрибут качества** | Usability |
| **Приоритет** | P0 |
| **Статус** | УТВЕРЖДЕНО |
| **Источник** | — |
| **Критерии приёмки** | — |

---


## What It Does

Реализация GUI согласно дизайну в `design/frontend.pen` и вызов методов API на клиенте. На стороне бэкенда допускаются заглушки (stubs) для API-вызовов.

## Scope

Milestone покрывает следующие экраны из `design/frontend.pen`:

### Auth
1. **Login Page** (`loginP`, 1920×1080) — страница входа через OAuth провайдеров: VK ID, Yandex ID, Mail.ru, Google, Corporate SSO (SAML/OIDC). Карточка с логотипом VEDO и 5 кнопками OAuth.

### Main Application (Header + Sidebar layout)
2. **Dashboard** (`dashB`, 1920×1080) — главная панель с приветствием (аватар, имя, роль, статус), коллаборационными виджетами (Merge Requests, Reviews, Work Items) с счётчиками и временем последней активности.

3. **Ontology Workspace** (`ontoWs`, 1920×1080) — 3-панельный редактор: Organism/ClassTree (слева, 400px), Organism/GraphVisualization (центр), Organism/PropertyPanel (справа, 400px). Верхняя панель: Organism/OntologyToolbar (хлебные крошки, Save/Undo/Redo/Validate). Боковая панель: Organism/GroupSidebar. Поддерживает переключение между GroupSidebar и SidebarCompact.

4. **SPARQL Query Builder** (`spqBld`, 1920×2260) — редактор SPARQL-запросов: Organism/SPARQLQueryEditor с синтаксической подсветкой, кнопками Run/Fmt, таблицей результатов.

5. **Metrics Dashboard** (`metDas`, 1920×2260, layout: none) — метрики и аналитика: Organism/MetricsDashboard с графиками онтологии (классы, свойства, индивиды, аксиомы, тренды).

6. **Public Ontology View** (`pubOnt`, 1920×1080) — публичный read-only просмотр онтологии. Custom publicHeader (без поиска, урезанный), Organism/ClassTree (без кнопки создания), Organism/GraphVisualization, Organism/PropertyPanel (все вкладки disabled). Бейдж "Read-only queries only".

### Members & Administration
7. **Members Panel** (`memPan`, 1920×1080) — управление участниками: Organism/MembersPanel с таблицей (аватар, имя, роль, дата добавления, действия).

### SHACL Validation
8. **SHACL Rule Builder** (`shaclR`, 1920×1080) — редактор SHACL-правил: Organism/SHACLRuleBuilder с деревом правил, редактором, таргет-селектором, condition builder, action picker.

9. **Validation Report** (`valRep`, 1920×1080) — отчёт SHACL валидации: кнопка Run Validation, Organism/ValidationReport с суммаризацией (pass/fail/warn) и таблицей (Rule/Severity/Focus/Message).

### Versioning (Git-like)
10. **Commits** (`ErMOb`, 1920×1080) — история коммитов: фильтры (ветка, автор, поиск), Organism/CommitHistory с таблицей (автор, сообщение, SHA, дата).

11. **Branches** (`joJk5`, 1920×1080) — управление ветками: Organism/BranchList с таблицей (ветка, коммит, обновлено, действия).

12. **Compare Revisions** (`u2YR0`, 1920×1080) — сравнение ревизий: селекторы base/target, diff stats, Organism/DiffView со сплит-диффом (added/removed/changed строки).

13. **Tags** (`A6l32P`, 1920×1080) — управление тегами: поиск, Organism/TagList с таблицей (имя, коммит, описание, обновлено, действия), пагинация, empty/loading states.

14. **Repository Graph** (`n3Hs7`, 1920×1080) — визуализация Git-графа: Organism/RepositoryGraph с DAG, узлами коммитов, ветками, тегами, тултипами.

15. **Merge Requests** (`G83Yfe`, 1920×1080) — страница Merge Requests. **Placeholder** — контентная область содержит только текст-заглушку, Organism ещё не реализован.

### Organism Components (21 шт.)

| Компонент | ID | Назначение |
|-----------|-----|-----------|
| Organism/Header | `qpgfz` | Верхняя панель: логотип VEDO + бренд, поиск, аватар пользователя, уведомления (bell), настройки (gear) |
| Organism/Sidebar | `W5vtj` | Основная навигация: Dashboard (active), 5 пунктов с иконками (Ontology Explorer, SPARQL Queries, Metrics, Members, Settings) |
| Organism/SidebarCompact | `TJOOr` | Узкая версия 72px с иконками |
| Organism/GroupSidebar | `SItfr` | Боковая панель группы: аватар группы, название, описание, навигация (Dashboard, Ontology, Members, Settings) |
| Organism/CollapsedGroupSidebar | `t3slf5` | Свёрнутая версия GroupSidebar (64px, только иконки) |
| Organism/OntologyToolbar | `cZ9F0` | Панель инструментов: хлебные крошки, Save/Undo/Redo/Validate, переключатели вида |
| Organism/ClassTree | `sl8av` | Дерево классов: поиск/фильтр, переключатель tree/table/graph, expandable узлы (owl:Thing, Person, Organization, Product) |
| Organism/PropertyPanel | `c0wXES` | Панель свойств: вкладки Properties, Restrictions, Usage, Annotations, Metadata |
| Organism/GraphVisualization | `RKIbA` | Визуализация графа: canvas с zoom, тулбар (layout, filters), таблица индивидов (Individual/Property/Value) |
| Organism/SPARQLQueryEditor | `spqlE` | SPARQL редактор: подсветка синтаксиса, кнопки Run/Fmt, таблица результатов |
| Organism/MetricsDashboard | `mtrD` | Метрики онтологии: счётчики классов/свойств/индивидов/аксиом, графики трендов, селектор онтологии |
| Organism/OntologyMetadata | `ontM` | Метаданные онтологии: заголовок, описание, версия, namespace |
| Organism/MembersPanel | `membP` | Управление участниками: таблица (аватар, имя, роль, дата, действия), бейдж количества |
| Organism/CommitHistory | `Mg9Hs` | История коммитов: бейдж ветки, таблица (Author/Message/SHA/Date) |
| Organism/DiffView | `c86ae7` | Сравнение версий: селекторы base/target, diff stats, сплит-дифф |
| Organism/BranchList | `tTpUr` | Список веток: segcontrol All/Active, таблица (Branch/Commit/Updated/Actions) |
| Organism/TagList | `u0Wa7Z` | Список тегов: поиск, таблица (Tag name, Commit, Description, Updated, Actions), пагинация, empty/loading |
| Organism/RepositoryGraph | `repoG` | Git DAG граф: узлы коммитов, ветки, теги, тултипы |
| Organism/CommentsThread | `cmnt` | Обсуждения: threaded комментарии с аватаром, текстом, reply, resolve. Через API Gateway WebSocket. |
| Organism/SHACLRuleBuilder | `shRB` | Редактор SHACL-правил: дерево правил, редактор, таргет-селектор, condition builder, action picker |
| Organism/ValidationReport | `valR` | Отчёт валидации: суммаризация pass/fail/warn, таблица (Rule/Severity/Focus/Message) |

### UI-Kit Components (28 импортированных типов)

Дизайн-система импортируется из `ui-kit.lib.pen` (B: refs). Ключевые компоненты:

B:c8T4z (OAuthButton), B:ztFq7 (Avatar/IconFrame), B:kLoW9 (GhostButton), B:3CI7c (PrimaryButton), B:K5sXB (Select), B:Pw7LN (Badge), B:FTaL1 (TextInput), B:cN7Wo (Checkbox), B:nCzv2 (Radio), B:aTMES (Dialog), B:NwwhS (Badge info), B:yA4l7 (SearchInput), B:ahk3w (IconButton), B:ck1WW (Tab), B:AOlSf (Divider), B:JDECb (ExpandableSection), B:j6AS7 (TableColumn), B:u0ezf (Spacer)

### Dialogs (8 шт.)

Диалоги расположены на отдельном canvas (`dialogS`, 2736×10844, layout: none):

1. **Create Class Dialog** — IRI (auto-generated), label (rdfs:label), comment (rdfs:comment), subclass-of select, abstract/deprecated
2. **Create Property Dialog** — Object/Datatype/Annotation radio, IRI, label, comment, domain/range select, OWL characteristics (functional, transitive, symmetric, inverse-functional, reflexive, irreflexive), inverse, min/max cardinality
3. **Create Individual Dialog** — IRI, label, types select, data properties (worksFor, age, email, height — add/remove)
4. **Annotation Dialog** — property select, language select, value field
5. **Import Dialog** — file upload, import strategy select, dry-run, progress bar, import plan table, replace alert
6. **MFA Challenge Dialog** — lock icon, info alert, code input, recovery button
7. **Publish Snapshot Dialog** — intro, info alert, branch field, name/description/URL inputs, copy button

### Design Tokens

Все токены дизайна импортируются из `ui-kit.lib.pen` (переменные `$B:*`):
- Цвета: `$B:background`, `$B:card`, `$B:border`, `$B:foreground`, `$B:muted-foreground`, `$B:primary`, `$B:primary-foreground`, `$B:popover`
- Шрифт: `IBM Plex Mono` (моноширинный, используется во всём дизайне)
- Иконки: Lucide (`iconFontFamily: "lucide"`)

## API Layer

- Реализовать вызовы методов API на клиенте через API Gateway
- На стороне бэкенда могут быть заглушки (stubs), возвращающие мок-данные
- Протоколы: GraphQL для навигации по графу, REST для CRUD-операций, WebSocket для уведомлений

## Relation to Collaboration Service

- Collaboration Service (Go) удалён
- Lock Conflict Indicator — удалён из дизайна, заменён Git-подобным версионированием
- CommentsThread — работает через API Gateway WebSocket, отдельный сервис не требуется
- Merge Requests — страница существует (placeholder), требует реализации Organism

## Покрытие экранов Organism-компонентами

| Экран | Header | Sidebar | Основные компоненты |
|-------|--------|---------|-------------------|
| Login Page | — | — | 5× OAuthButton |
| Dashboard | O/Header | O/Sidebar | Greeting card, 3 collab widgets |
| Ontology Workspace | O/Header | O/GroupSidebar | O/OntologyToolbar, O/ClassTree, O/GraphVisualization, O/PropertyPanel |
| SPARQL Query Builder | O/Header | O/Sidebar | O/SPARQLQueryEditor |
| Metrics Dashboard | O/Header | O/Sidebar | O/MetricsDashboard |
| Public Ontology View | publicHeader (custom) | — | O/ClassTree (disabled create), O/GraphVisualization, O/PropertyPanel (disabled) |
| Members Panel | O/Header | O/Sidebar | O/MembersPanel |
| SHACL Rule Builder | O/Header | O/Sidebar | O/SHACLRuleBuilder |
| Validation Report | O/Header | O/Sidebar | Run btn + O/ValidationReport |
| Commits | O/Header | O/Sidebar | Filters + O/CommitHistory |
| Branches | O/Header | O/Sidebar | O/BranchList |
| Compare Revisions | O/Header | O/Sidebar | O/DiffView |
| Tags | O/Header | O/Sidebar | O/TagList |
| Repository Graph | O/Header | O/Sidebar | O/RepositoryGraph |
| Merge Requests | O/Header | O/Sidebar | placeholder text |

## Error Cases
- Ошибки сети при вызовах API
- Неподдерживаемый браузер
- Ошибки GraphQL/REST запросов (обработка на клиенте)

## Business Rules
- Все экраны должны соответствовать дизайну в frontend.pen
- API-вызовы должны быть готовы к подключению реального бэкенда (заглушки заменяются на реальные вызовы)
- Компоненты переиспользуются между экранами
- Merge Requests (`G83Yfe`) требует реализации Organism (текущий placeholder)
