# F3: Git-подобный контроль версий (Versioning Service)

| Атрибут | Значение |
|---------|----------|
| **ID** | REQ-FUN.DATA.versioning |
| **Уровень** | FUN |
| **Атрибут качества** | Functionality |
| **Приоритет** | P0 |
| **Статус** | УТВЕРЖДЕНО |
| **Источник** | — |
| **Критерии приёмки** | — |

---

## Обзор

Встроенная система контроля версий для онтологий. Поддерживает два режима работы:

1. **Нативный Versioning Service (Rust)** — основной режим
2. **Git Backend** — опциональный режим для совместимости с CI/CD

**Почему нативный сервис (основной):**
1. Нестабильная сериализация RDF/OWL в Git → ложные diff
2. Git не подходит для ABox с миллионами индивидов
3. Git не поддерживает семантическое слияние OWL-конструкций

**Когда использовать Git:**
- Интеграция с существующими Git workflow
- CI/CD пайплайны, которые анализируют изменения онтологии
- Совместимость с GitHub/GitLab (PR, ревью кода)
- Организации с уже устоявшимися Git-процессами

## Архитектура

```
┌─────────────────────────────────────────────────────────────────┐
│                      Versioning Service (Rust)                  │
├─────────────────────────────────────────────────────────────────┤
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐              │
│  │ Commit      │  │ Branch      │  │ Diff        │              │
│  │ Manager     │  │ Manager     │  │ Engine      │              │
│  └─────────────┘  └─────────────┘  └─────────────┘              │
│                                                              │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐              │
│  │ Turtle      │  │ Merge       │  │ Conflict    │              │
│  │ Canonical   │  │ Resolver    │  │ Detector    │              │
│  └─────────────┘  └─────────────┘  └─────────────┘              │
│                                                              │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │              Storage Backend (pluggable)                │    │
│  │  ┌─────────────────┐    ┌─────────────────┐            │    │
│  │  │  Native Store    │    │  Git Backend    │            │    │
│  │  │  (PostgreSQL)    │    │  (libgit2)      │            │    │
│  │  └─────────────────┘    └─────────────────┘            │    │
│  └─────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────┘
```

## Выбор backend

```yaml
versioning:
  backend: native  # or 'git'
  
  # Для native backend:
  native:
    commit_store: postgresql
    branch_store: postgresql
    cache: redis
  
  # Для git backend:
  git:
    repository_path: /data/repos/{ontologyId}
    author_mapping:   # как Map userId → Git author
      "user-123":
        name: "Nikolay Marusov"
        email: "nikolay@vedo.ai"
    push_url: ""     # remote для автопуша (опционально)
    hooks:
      pre_commit: ""   # скрипт валидации перед коммитом
      post_commit: ""  # webhook после коммита
```

## Сравнение режимов

| Возможность | Native (Rust) | Git |
|---------|--------------|-----|
| Семантический diff | ✅ С учетом OWL | ❌ Только синтаксический |
| Большой ABox | ✅ Оптимизировано | ⚠️ Требуется LFS |
| Auto-merge | ✅ Умный | ❌ Ручной |
| Модель веток | ✅ Кастомная | ✅ Нативная Git |
| Интеграция CI/CD | ⚠️ Webhook | ✅ Нативная |
| Экосистема Git | ❌ Кастомная | ✅ PR, review, CI |
| Производительность | ✅ Быстро | ⚠️ Медленно для больших объемов |
| Каноническая сериализация | ✅ Встроена | ⚠️ Нужна предобработка |

## Native Backend (по умолчанию)

См. основной документ: Commit, Branch, Diff Engine, Merge.

Метрики качества конфликтов и ложных конфликтов для merge измеряются по `human/artifacts/requirements/REQ-FUN.INTEGRATION.collaboration-quality-metrics.md`.

## Git Backend

### Как это работает

```
User commits
    │
    ▼
Versioning Service (Rust)
    │
    ├── Serialize ontology to Turtle (canonical)
    │
    ├── Write to Git working directory
    │   └── /data/repos/{ontologyId}/ontology.ttl
    │
    ├── Stage: git add ontology.ttl
    │
    └── Commit: git commit -m "message"
                --author="Name <email>"
                --date="ISO8601"
    
    │
    ▼ (optional)
Git remote (GitHub/GitLab)
```

### Операции Git

```typescript
interface GitBackend {
  // Initialize repo for ontology
  initRepo(ontologyId: string): Promise<void>
  
  // Stage + commit changes
  commit(
    ontologyId: string,
    delta: TurtleDelta,
    message: string,
    author: GitAuthor
  ): Promise<GitCommit>
  
  // Checkout specific version
  checkout(ontologyId: string, ref: string): Promise<OntologyState>
  
  // Get diff between commits
  diff(from: string, to: string): Promise<string>
  
  // Branch operations
  createBranch(name: string): Promise<void>
  listBranches(): Promise<string[]>
  
  // Merge (manual resolution)
  merge(source: string, target: string): Promise<MergeResult>
  
  // Export/Import
  push(): Promise<void>
  pull(): Promise<void>
}
```

### Предобработка Turtle для Git

Git работает с файлами, поэтому нужна предобработка:

```rust
fn serialize_for_git(ontology: &Ontology) -> String {
    // 1. Каноническая сериализация Turtle
    let turtle = ontology.to_turtle_canonical();
    
    // 2. Нормализация порядка
    let normalized = normalize_turtle(&turtle);
    
    // 3. Добавление метаданных коммита в заголовок
    let with_meta = format!(
        "# VEDO Commit: {}\n# Author: {}\n# Date: {}\n# Message: {}\n\n{}",
        commit_id,
        author,
        date,
        message,
        normalized
    );
    
    return with_meta;
}
```

### Пример Git log

```bash
$ git log --oneline
a1b2c3d Add ThermalPowerPlant class
e4f5g6h Update energy object properties
i7j8k9l Initial ontology structure

$ git show a1b2c3d --stat
 a1b2c3d |  234 +45 -12
 1 file changed, 234 insertions, 12 deletions
```

### Интеграция CI/CD

```yaml
# .github/workflows/ontology-ci.yml
name: Ontology Validation

on:
  push:
    branches: [main, develop]
  pull_request:

jobs:
  validate:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          path: ontology
      
      - name: Validate Turtle syntax
        run: |
         rapper -i turtle -o ntriples ontology/ontology.ttl
      
      - name: Check OWL DL consistency
        run: |
          # Использовать external reasoner
          java -jar orizza.jar validate ontology/ontology.ttl
      
      - name: Generate diff report
        if: github.event_name == 'pull_request'
        run: |
          python scripts/diff_report.py ${{ github.event.pull_request.base.sha }} HEAD

      - name: Comment PR with changes
        uses: actions/github-script@v7
        with:
          script: |
            // Post diff summary as PR comment
```

### Sync Native ↔ Git

```rust
enum SyncDirection {
    PushNativeToGit,
    PullGitToNative,
    Bidirectional,
}

async fn sync_ontology(
    ontology_id: &str,
    direction: SyncDirection,
    endpoint: &str
) -> Result<SyncReport> {
    match direction {
        PushNativeToGit => {
            // Экспорт из PostgreSQL → Git commit
            let state = native_store.get_state(ontology_id)?;
            let turtle = serialize_for_git(&state);
            git_backend.write_file("ontology.ttl", &turtle)?;
            git_backend.commit("Sync from native", author)?;
        }
        PullGitToNative => {
            // Git checkout → Import в PostgreSQL
            let turtle = git_backend.read_file("ontology.ttl")?;
            let state = parse_ontology(&turtle)?;
            native_store.restore_state(ontology_id, state)?;
        }
        Bidirectional => {
            // Merge с приоритетом (custom rules)
            unimplemented!("Need conflict resolution strategy")
        }
    }
}
```

### Когда использовать каждый режим

| Сценарий | Рекомендуемый backend |
|----------|-------------------|
| Небольшая команда, быстрые итерации | Native |
| Большая онтология (1M+ триплетов) | Native |
| Существующий Git workflow | Git |
| CI/CD pipelines требуют diffs | Git |
| Сложные merge conflicts | Native |
| Команда знакома с Git | Git |
| Нужен семантический merge | Native |

### Миграция между backend

```typescript
// Export from Native → Import to Git
async function migrateToGit(ontologyId: string): Promise<void> {
  const commits = await native.getCommits(ontologyId)
  
  git.initRepo(ontologyId)
  
  for (const commit of commits) {
    const state = await native.getState(ontologyId, commit.id)
    const turtle = serializeForGit(state)
    
    git.writeFile('ontology.ttl', turtle)
    git.add()
    git.commit(commit.message, commit.author)
  }
  
  // Опционально: push to remote
  if (config.git.pushUrl) {
    git.push()
  }
}
```

## API (унифицированный)

API унифицирован — клиент не знает, какой backend используется:

```graphql
type Query {
  # Работает одинаково для обоих backends
  commits(ontologyId: ID!, branch: String, limit: Int): CommitConnection!
  commit(id: ID!): Commit
  branches(ontologyId: ID!): [Branch!]!
  diff(from: ID!, to: ID!): Diff!
}

type Mutation {
  createCommit(ontologyId: ID!, message: String!): Commit!
  createBranch(ontologyId: ID!, name: String!, sourceBranch: String): Branch!
  merge(ontologyId: ID!, sourceBranch: String!, targetBranch: String!): MergeResult!
}

type BackendInfo {
  type: 'native' | 'git'
  repositoryUrl: String       # Только для git
  lastSync: DateTime
  syncStatus: 'synced' | 'pending' | 'error'
}
```

## Сравнение хранения

### Native Backend

| Данные | Хранилище |
|------|---------|
| Метаданные коммитов | PostgreSQL |
| Указатели веток | PostgreSQL |
| Diff коммитов | PostgreSQL (JSONB) |
| Состояние онтологии | Neo4j |
| Кеш/блокировки | Redis |

### Git Backend

| Данные | Хранилище |
|------|---------|
| Файлы онтологии | Репозиторий Git (файловая система) |
| Коммиты | Git objects |
| Ветки | Git refs |
| Метаданные | Git notes или sidecar JSON |

## Ключевые понятия

### 1. Commit

**Что фиксируется:**
- Дельта изменений (diff)
- Автор (user_id, keycloak token)
- Сообщение коммита
- Временная метка
- Родительский коммит

**Структура данных:**

```typescript
interface Commit {
  id: string;              // UUID
  ontologyId: string;     // Онтология
  branchId: string;       // Ветка
  parentIds: string[];     // Родители (1 для normal, 2+ для merge)
  
  author: {
    userId: string;
    displayName: string;
  };
  
  message: string;         // Коммит-месседж
  timestamp: Date;
  
  // Сериализация дельты
  delta: {
    added: NodeChange[];   // Классы, свойства, индивиды
    removed: NodeChange[];
    modified: PropertyChange[];
  };
  
  // Метаданные
  stats: {
    nodesAdded: number;
    nodesRemoved: number;
    nodesModified: number;
  };
}

interface NodeChange {
  type: 'Class' | 'ObjectProperty' | 'DatatypeProperty' | 'Individual';
  id: string;
  before?: SerializedNode;  // Для removed/modified
  after?: SerializedNode;   // Для added/modified
}

interface PropertyChange {
  nodeId: string;
  property: string;
  before: string | string[];
  after: string | string[];
}
```

### 2. Branch

```typescript
interface Branch {
  id: string;
  name: string;            // human-readable: "main", "feature/new-class"
  ontologyId: string;
  
  head: string;            // commitId последнего коммита
  createdAt: Date;
  createdBy: string;
  
  isProtected: boolean;    // protected branches нельзя удалять
  mergeStrategy: 'fast-forward' | 'merge-commit';
}
```

**Именование веток:**
- `main` — основная ветка (protected)
- `dev` — разработка (default для новых пользователей)
- `feature/*` — фичи
- `hotfix/*` — срочные исправления

### 3. Diff Engine

**Каноническая сериализация Turtle:**
- Алфавитная сортировка триплетов
- Стабильный порядок элементов
- Идентичные онтологии → идентичные файлы

**Типы diff:**
```typescript
type DiffType = 'semantic' | 'syntactic' | 'stats';

interface SemanticDiff {
  addedClasses: Class[];
  removedClasses: Class[];
  modifiedClasses: {
    id: string;
    changes: PropertyChange[];
  }[];
  addedProperties: Property[];
  removedProperties: Property[];
  // ... для каждого типа сущностей
}
```

## API-эндпоинты (Git-like)

API максимально повторяет структуру GitHub/GitLab API.

### Структура базового URL

```
/api/v1/repos/{ontologyId}/git/
```

### 1. References (ветки)

Аналог: `git branch` / GitHub `/repos/{owner}/{repo}/branches`

```
GET    /git/refs
  → List all branches (refs/heads/*)

GET    /git/refs/{branch}
  → Get branch info + current commit SHA
  Response: { ref: "refs/heads/main", object: { sha: "abc123" } }

POST   /git/refs
  Body: { ref: "refs/heads/feature/new-class", sha: "abc123" }
  → Create branch at commit
  Alias: git branch feature/new-class abc123

DELETE /git/refs/{branch}
  → Delete branch
  Alias: git branch -d feature/new-class
```

### 2. Commits

Аналог: `git log` / GitHub `/repos/{owner}/{repo}/commits`

```
GET    /git/commits
  Query: ?ref=main&path=&sha=HEAD&since=&until=&author=
  → List commits
  Headers: Link (pagination), ETag
  Response: [{ sha, commit: { message, author, committer, tree } }, ...]

  Alias: git log [--author=<author>] [--since=<date>]

GET    /git/commits/{sha}
  → Get single commit with full details
  Response: { sha, message, parents[], tree, stats, files }

  Alias: git show <sha>

POST   /git/commits
  Body: {
    message: "Add new class",
    author: { name: "Nikolay", email: "nikolay@vedo.ai", date: "2026-05-09T..." },
    parents: ["sha-of-parent"],
    tree: { [path]: { sha: "blob-sha" } }  // для native
  }
  → Create commit
  Alias: git commit -m "message"

GET    /git/commits/{sha}/diff/{baseSha}
  → Diff between commits
  Query: ?unified=3
  Response: { files: [{ path, status, additions, deletions, patch }] }
  Alias: git diff <baseSha> <sha>
```

### 3. Trees (файловая структура)

Аналог: `git ls-tree` / GitHub `/repos/{owner}/{repo}/git/trees`

```
GET    /git/trees/{sha}
  Query: ?recursive=1
  → Get tree (ontology snapshot)
  Response: {
    sha: "tree-sha",
    tree: [
      { path: "ontology.ttl", mode: "100644", type: "blob", sha: "blob-sha" },
      { path: "metadata.json", mode: "100644", type: "blob", sha: "blob-sha" }
    ]
  }
  Alias: git ls-tree -r <sha>

GET    /git/trees/{sha}?recursive=1&filter=classes
  → Get subtree filtered by entity type
  Filter options: classes, properties, individuals, annotations
```

### 4. Blobs (содержимое файлов)

Аналог: `git cat-file -p` / GitHub `/repos/{owner}/{repo}/git/blobs/{sha}`

```
GET    /git/blobs/{sha}
  Query: ?encoding=turtle|rdf-xml
  → Get file content
  Response: { sha, content: "...", encoding: "base64" }
  Alias: git cat-file -p <sha>

POST   /git/blobs
  Body: { content: "...", encoding: "utf-8" }
  → Create blob (staging)
  Response: { sha: "blob-sha" }
  Alias: git hash-object -w --stdin
```

### 5. Status и Stash (рабочая директория)

Аналог: `git status` / `git stash`

```
GET    /git/status
  → Current working tree status
  Response: {
    branch: "main",
    sha: "abc123",
    ahead: 0,
    behind: 0,
    staged: [ { path: "ontology.ttl", status: "modified" } ],
    unstaged: [ { path: "metadata.json", status: "added" } ],
    untracked: []
  }
  Alias: git status

POST   /git/stash
  Body: { message: "WIP: work in progress" }
  → Stash uncommitted changes
  Alias: git stash

GET    /git/stash
  → List stashes
  Alias: git stash list

GET    /git/stash/{stashId}
  → Get stash content
  Alias: git stash show

POST   /git/stash/{stashId}/apply
  → Apply stash
  Alias: git stash apply

DELETE /git/stash/{stashId}
  → Drop stash
  Alias: git stash drop
```

### 6. Теги

Аналог: `git tag`

```
GET    /git/tags
  → List tags
  Alias: git tag -l

POST   /git/tags
  Body: { tag: "v1.0.0", message: "Release v1.0", sha: "abc123" }
  → Create tag
  Alias: git tag -a v1.0.0 -m "Release v1.0" abc123

GET    /git/tags/{tag}
  → Get tag info
  Alias: git show v1.0.0

DELETE /git/tags/{tag}
  → Delete tag
  Alias: git tag -d v1.0.0
```

### 7. Merge и Rebase

Аналог: `git merge` / `git rebase`

```
POST   /git/merges
  Body: {
    base: "main",
    head: "feature/new-class",
    message: "Merge feature/new-class into main",
    strategy: "auto" | "manual"
  }
  → Merge branch
  Alias: git merge feature/new-class
  Response: { sha, merge_type: "fast-forward" | "merge-commit", conflicts?: [...] }

GET    /git/merges/{mergeId}/conflicts
  → Get merge conflicts
  Response: { files: [{ path, ours, theirs, merged? }] }

PUT    /git/merges/{mergeId}/resolve
  Body: { resolutions: [{ path, resolution: "ours" | "theirs" | "manual", content? }] }
  → Resolve conflicts
  Alias: git add <resolved-files> && git commit

POST   /git/rebase
  Body: { base: "main", head: "feature/new-class" }
  → Rebase branch
  Alias: git rebase main
```

### 8. Diff

Аналог: `git diff` / `git difftool`

```
GET    /git/compare/{base}...{head}
  → Compare two commits/branches/refs
  Response: {
    status: "ahead" | "behind" | "diverged" | "identical",
    ahead_by: 2,
    behind_by: 3,
    commits: [...],
    files: [{ path, status, additions, deletions, patch }]
  }
  Alias: git log main..feature

GET    /git/diff/{sha1}:{path}..{sha2}:{path}
  → Diff specific file between commits
  Alias: git diff abc123:ontology.ttl def456:ontology.ttl
```

### 9. Archive и Export

Аналог: `git archive`

```
GET    /git/archives/{ref}
  Query: ?format=tar.gz|zip
  → Download ontology as archive
  Alias: git archive -o archive.tar.gz HEAD

GET    /git/export/{format}
  Query: ?format=turtle|rdf-xml|n-triples
  → Export ontology in specific RDF format
  Alias:riot --format=TURTLE < ontology.ttl

POST   /git/import
  Body: { file: <multipart>, branch: "main", message: "Import from file" }
  → Import ontology from file
  Alias: git add . && git commit
```

## REST API (упрощенный внешний фасад API Gateway)

Для обратной совместимости также доступен упрощённый REST на API Gateway. Versioning Service не публикует функциональный REST API; API Gateway транслирует эти маршруты во внутренние gRPC/protobuf вызовы Versioning Service.

```
# Commits
POST   /api/v1/ontologies/{id}/commits
GET    /api/v1/ontologies/{id}/commits
GET    /api/v1/ontologies/{id}/commits/{sha}

# Branches
GET    /api/v1/ontologies/{id}/branches
POST   /api/v1/ontologies/{id}/branches
DELETE /api/v1/ontologies/{id}/branches/{name}

# Merge
POST   /api/v1/ontologies/{id}/merge
GET    /api/v1/ontologies/{id}/merge/{id}/conflicts

# History
GET    /api/v1/ontologies/{id}/history
POST   /api/v1/ontologies/{id}/rollback

# Export/Import
GET    /api/v1/ontologies/{id}/export
POST   /api/v1/ontologies/{id}/import
```

## Сопоставление команд Git

| Команда Git | VEDO API |
|------------|----------|
| `git init` | `POST /git/repos` |
| `git clone` | `GET /git/clone/{ontologyId}` |
| `git status` | `GET /git/status` |
| `git add .` | Неявно (auto-stage) |
| `git commit -m "msg"` | `POST /git/commits` |
| `git log` | `GET /git/commits` |
| `git show <sha>` | `GET /git/commits/{sha}` |
| `git branch` | `GET /git/refs` |
| `git branch <name>` | `POST /git/refs` |
| `git checkout <branch>` | `GET /git/checkout/{ref}` |
| `git merge <branch>` | `POST /git/merges` |
| `git diff <a> <b>` | `GET /git/compare/{a}...{b}` |
| `git stash` | `POST /git/stash` |
| `git tag v1.0` | `POST /git/tags` |
| `git archive` | `GET /git/archives/{ref}` |

## GraphQL API

```graphql
type Query {
  commits(ontologyId: ID!, branch: String, limit: Int, offset: Int): CommitConnection!
  commit(id: ID!): Commit
  branches(ontologyId: ID!): [Branch!]!
  diff(from: ID!, to: ID!): Diff!
}

type Mutation {
  createCommit(ontologyId: ID!, message: String!): Commit!
  createBranch(ontologyId: ID!, name: String!, sourceBranch: String): Branch!
  deleteBranch(ontologyId: ID!, name: String!): Boolean!
  merge(ontologyId: ID!, sourceBranch: String!, targetBranch: String!): MergeResult!
  resolveConflicts(mergeId: ID!, resolutions: [ConflictResolution!]!): Commit!
  rollback(ontologyId: ID!, commitId: ID!): Commit!
}

type Subscription {
  commitCreated(ontologyId: ID!, branch: String): Commit!
  mergeStatusChanged(mergeId: ID!): MergeStatus!
}

type CommitConnection {
  edges: [CommitEdge!]!
  pageInfo: PageInfo!
  totalCount: Int!
}

type Commit {
  id: ID!
  message: String!
  author: User!
  timestamp: DateTime!
  parentIds: [ID!]!
  stats: CommitStats!
  diff: Diff
}
```

## UI-компоненты

### 1. Панель версий

```
┌──────────────────────────────────────────┐
│ Versions                          🔍    │
├──────────────────────────────────────────┤
│ Branch: [main ▼]  [+ New Branch]         │
├──────────────────────────────────────────┤
│ ┌──────────────────────────────────────┐ │
│ │ ○ Initial commit          Nikolay     │ │
│ │ ● Add Energy class         Marusov    │ │
│ │ │                          2026-05-09 │ │
│ │ │                          +12 -3     │ │
│ │ ○ Update properties         ...       │ │
│ └──────────────────────────────────────┘ │
├──────────────────────────────────────────┤
│ [Commit] [History] [Compare] [Export]    │
└──────────────────────────────────────────┘
```

### 2. Модальное окно commit

```
┌──────────────────────────────────────────┐
│ Commit changes                        ✕  │
├──────────────────────────────────────────┤
│ Message:                                 │
│ ┌──────────────────────────────────────┐ │
│ │ Add ThermalPowerPlant class           │ │
│ │ + define domain/range for properties  │ │
│ └──────────────────────────────────────┘ │
│                                          │
│ Changes:                                 │
│ ┌──────────────────────────────────────┐ │
│ │ A  Class: ТепловаяЭлектростанция     │ │
│ │ M  Class: ЭнергетическийОбъект        │ │
│ │ A  Property: hasCapacity (xsd:double) │ │
│ └──────────────────────────────────────┘ │
│                                          │
│ [Cancel]                    [Commit]      │
└──────────────────────────────────────────┘
```

### 3. Список веток

```
┌──────────────────────────────────────────┐
│ Branches                                 │
├──────────────────────────────────────────┤
│ ● main          (protected)   HEAD       │
│   dev           (default)     HEAD       │
│   feature/new-class              +3      │
│   feature/api-doc                -1 +2   │
├──────────────────────────────────────────┤
│ [Create Branch] [Merge] [Delete]        │
└──────────────────────────────────────────┘
```

### 4. Просмотр diff

```
┌──────────────────────────────────────────┐
│ Compare: v1 → v2                        ✕  │
├──────────────────────────────────────────┤
│ ┌─ Added (green) ─────────────────────┐   │
│ │ + Class: ТепловаяЭлектростанция    │   │
│ │ +   rdfs:subClassOf: ЭнергОбъект    │   │
│ │ + Property: hasCapacity            │   │
│ │     range: xsd:double              │   │
│ └────────────────────────────────────┘   │
│ ┌─ Modified (yellow) ────────────────┐    │
│ │ ~ Class: ЭнергетическийОбъект      │   │
│ │     comment: "Объект..." → "..."   │   │
│ └────────────────────────────────────┘   │
│ ┌─ Removed (red) ────────────────────┐    │
│ │ - Property: legacyProperty         │    │
│ └────────────────────────────────────┘   │
└──────────────────────────────────────────┘
```

### 5. Разрешение конфликтов

```
┌──────────────────────────────────────────┐
│ Resolve conflicts (3)                   ✕  │
├──────────────────────────────────────────┤
│ Conflict #1: Class modified              │
│ ┌──────────────────────────────────────┐ │
│ │ Class: ЭнергетическийОбъект          │ │
│ │                                      │ │
│ │ Their version (feature/new):         │ │
│ │   label: "Energy Object"             │ │
│ │   comment: "Base class for energy..." │ │
│ │                                      │ │
│ │ Our version (main):                  │ │
│ │   label: "Энергетический объект"     │ │
│ │   comment: "Базовый класс..."        │ │
│ │                                      │ │
│ │ Resolution:                          │ │
│ │ ( ) Use Their version                │ │
│ │ ( ) Use Our version                  │ │
│ │ (•) Manual merge:                    │ │
│ │   ┌────────────────────────────┐    │ │
│ │   │ label: "Энерг. объект"       │    │ │
│ │   └────────────────────────────┘    │ │
│ └────────────────────────────────────┘ │
├──────────────────────────────────────────┤
│ [Cancel]               [Resolve All]    │
└──────────────────────────────────────────┘
```

## События WebSocket

```typescript
// Для realtime collaboration
type VersioningEvent = 
  | { type: 'commit_created', commit: Commit }
  | { type: 'branch_created', branch: Branch }
  | { type: 'merge_started', mergeId: string }
  | { type: 'merge_completed', result: MergeResult }
  | { type: 'conflict_detected', conflicts: Conflict[] }
```

## Хранение

### Схема PostgreSQL

```sql
CREATE TABLE commits (
  id UUID PRIMARY KEY,
  ontology_id UUID NOT NULL,
  branch_id UUID NOT NULL,
  parent_ids UUID[] NOT NULL,
  author_id VARCHAR(255) NOT NULL,
  author_name VARCHAR(255),
  message TEXT NOT NULL,
  timestamp TIMESTAMPTZ DEFAULT NOW(),
  delta JSONB NOT NULL,
  stats JSONB
);

CREATE TABLE branches (
  id UUID PRIMARY KEY,
  name VARCHAR(255) NOT NULL,
  ontology_id UUID NOT NULL,
  head_commit_id UUID REFERENCES commits(id),
  is_protected BOOLEAN DEFAULT FALSE,
  created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_commits_ontology ON commits(ontology_id);
CREATE INDEX idx_commits_branch ON commits(branch_id);
CREATE INDEX idx_branches_ontology ON branches(ontology_id);
```

### Кеш Redis

- Указатели head веток
- Diff коммитов (hot commits)
- Токены блокировок для merge operations

### Производительность

| Операция | Цель |
|-----------|--------|
| Создание коммита | < 100ms |
| Список коммитов (100) | < 50ms |
| Вычисление diff | < 200ms |
| Auto-merge | < 500ms |
| Checkout ветки | < 1s |

### Политика retention

| Окружение | TBox | ABox (Active) | ABox (Archived) | LFS Binary |
|-----------|------|---------------|----------------|------------|
| **Sandbox** | 1000 коммитов | 500 коммитов | 14 дней | 30 дней |
| **Staging** | 10,000 коммитов | 2,000 коммитов | 90 дней | 90 дней |
| **Production** | ∞ | ∞ | 1 год | 1 год |

### Стратегия backup

| Тип | Частота | Retention | RTO | RPO |
|-----|---------|-----------|-----|-----|
| Git push (репликация) | Real-time | 1 год | 1 мин | 0 сек |
| Snapshot LFS | 1 час | 90 дней | 15 мин | 5 сек |
| PostgreSQL дамп | 1 день | 30 дней | 30 мин | 24 часа |
| Архив Glacier | 7 дней | 7 лет | 12 часов | 7 дней |

### Квоты (по умолчанию)

| Тип | Размер | Лимит сущностей |
|-----|--------|------------------|
| TBox | 100 MB | 10,000 классов |
| ABox | 10 GB | 100,000 индивидов |
| LFS | 50 GB | 1,000 файлов |

<!-- Все открытые вопросы закрыты — см. раздел с ответами на открытые вопросы выше -->

### 1. Стратегия sync при bidirectional mode

**Стратегия: Last-Write-Wins + семантический мерж**

| Сценарий | Стратегия | Приоритет |
|----------|-----------|-----------|
| **Конфликт без зависимостей** | LWW (Last Write Wins) | Последний коммит по времени |
| **Конфликт с зависимостями** | Семантический мерж | Тот, кто разрешил зависимости |
| **Неразрешимый конфликт** | Ручной merge | Пользователь |

```rust
struct Commit {
    id: String,
    ontology_id: String,
    timestamp: u64,  // Unix timestamp (наносекунды)
    vector_clock: HashMap<String, u64>,  // { "server_id": counter }
}

impl Commit {
    fn is_newer_than(&self, other: &Commit) -> bool {
        // 1. Сравнение векторных часов
        if let Some(relation) = self.compare_vector_clock(other) {
            return matches!(relation, ClockRelation::After);
        }
        // 2. Fallback на timestamp
        self.timestamp > other.timestamp
    }
}
```

### 2. Git LFS для больших ABox

| Параметр | Значение | Обоснование |
|----------|----------|-------------|
| **Макс. размер файла** | 5 GB | GitHub 2GB, GitLab 5GB |
| **Порог для LFS** | 5 MB | Авто для файлов >5 MB |
| **Общий LFS лимит** | 50 GB (начальный) | Зависит от плана |

```bash
# .gitattributes
*.ttl filter=lfs diff=lfs merge=lfs -text
*.rdf filter=lfs diff=lfs merge=lfs -text
*.owl filter=lfs diff=lfs merge=lfs -text
*.parquet filter=lfs diff=lfs merge=lfs -text
```

### 3. Git hooks — кто управляет

| Тип | Кто управляет | Распространение |
|-----|---------------|-----------------|
| **Server-side** | Администратор | На сервере |
| **Client-side (core)** | Team Leader | `vedo init` |
| **Client-side (custom)** | Разработчик | Локально |

```bash
# Установка рекомендованных hooks
vedo hooks install --type=recommended
```

### 4. Auto-prune старых коммитов

| Тип данных | Retention | Стратегия |
|------------|-----------|-----------|
| **TBox (коммиты)** | ∞ | Никогда не удалять |
| **ABox (данные)** | 90 дней | `git gc` |
| **Binary LFS** | 30 дней | `git lfs prune` |

```bash
# Настройка
git config gc.pruneexpire "90 days ago"
git config lfs.pruneoffsetdays 30

# Cron job
0 2 * * 0 git gc --aggressive && git lfs prune
```

### 5. Squash commits

**Поддерживается:**

```typescript
// API
POST /api/v1/ontologies/{id}/commits/squash
{
    "commit_ids": ["abc123", "def456", "ghi789"],
    "new_message": "Refactor class hierarchy"
}
```

**Когда использовать:**
- ✅ Feature branch перед merge
- ✅ Мелкие фиксы (typo, formatting)
- ❌ Независимые изменения (теряется контекст)

### 6. Cherry-pick

**Поддерживается:**

```typescript
POST /api/v1/ontologies/{id}/cherry-pick
{
    "source_branch": "feature/new-ontology",
    "target_branch": "main",
    "commits": ["abc123", "def456"]
}
```

**Когда использовать:**
- ✅ Backport bug fix в старую версию
- ✅ Перенос одного класса из экспериментальной ветки
- ❌ Перенос целого модуля (10+ коммитов) → `git merge`

### 7. Binary data

**Многоуровневое хранение:**

| Тип | Размер | Решение |
|-----|--------|---------|
| **Изображения** | < 10 MB | Git LFS |
| **Эмбеддинги LLM** | 100 MB - 2 GB | Git LFS |
| **Дампы ABox** | 100 MB - 50 GB | DVC или Object Storage |
| **Видео/аудио** | > 10 MB | Object Storage (S3) + ссылка |

**DVC для больших данных:**

```bash
dvc init
dvc remote add -d myremote s3://vedo-data/
dvc add data/embeddings/model.pt
git add data/embeddings/model.pt.dvc
```
