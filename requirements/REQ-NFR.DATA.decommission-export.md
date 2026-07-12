# Экспорт при выводе из эксплуатации

| Атрибут | Значение |
|---------|----------|
| **ID** | REQ-NFR.DATA.decommission-export |
| **Уровень** | NFR |
| **Атрибут качества** | Functionality |
| **Приоритет** | P0 |
| **Статус** | УТВЕРЖДЕНО |
| **Источник** | — |
| **Критерии приёмки** | — |

---

## Что описывает

Фиксирует ответ на вопрос D1.1: в каком формате экспортируются данные при выводе VEDO Core из эксплуатации.

Вывод системы из эксплуатации должен происходить контролируемо, без потери данных и с возможностью дальнейшего использования данных в других системах. VEDO Core предоставляет стандартизированные открытые форматы, не привязанные к проприетарным решениям.

Главный принцип: всё, что вошло в VEDO, должно иметь возможность выйти без потери смысла.

## Форматы экспорта по типам данных

| Тип данных | Формат экспорта | Пригодность для импорта в другие системы | Человекочитаемость |
|------------|-----------------|------------------------------------------|--------------------|
| TBox: схема онтологии, классы, свойства, аксиомы | Канонический Turtle (`.ttl`) | Импортируется в Protégé, TopBraid, GraphDB, Apache Jena | Да |
| ABox: индивиды, факты | Turtle + JSON-LD | Turtle универсален, JSON-LD подходит для web API | Да |
| Version Store: коммиты, ветки, история | Git repository + JSON logs | Git стандартен, JSON подходит для аналитики | Да для Git |
| Audit logs | JSON Lines (`.jsonl`) + NDJSON | JSON универсален, подходит для SIEM и аналитики | Да |
| LFS-объекты: бинарные файлы | Сырые файлы в исходном формате | Да, без конвертации | Нет |
| Метрики и настройки: конфигурация, RBAC | YAML + JSON | Да | Да |

## Лимиты объёма и времени экспорта

| Формат | Макс. объём | Макс. время | Примечание |
|--------|-------------|-------------|------------|
| Canonical Turtle (.ttl) | 5 GB | 30 минут | Для TBox + ABox |
| JSON-LD | 5 GB | 30 минут | Альтернативный формат |
| Version Store (Git bundle) | 2 GB | 20 минут | Коммиты, ветки |
| Audit logs (JSON-L) | 1 GB | 15 минут | |
| Полный экспорт tenant | 10 GB (суммарно) | 90 минут | Все форматы в одном архиве |

Для объёмов выше указанных — экспорт выполняется асинхронно с уведомлением (email + UI). Ссылка на скачивание действительна 7 дней. Для on-premise параметры могут быть изменены конфигурацией, но VEDO не гарантирует время при превышении указанных объёмов.

## Экспорт TBox

TBox экспортируется в канонический Turtle.

Причины выбора Turtle вместо RDF/XML или OWL/XML:

- Turtle человекочитаем и его можно инспектировать в текстовом редакторе.
- Turtle компактнее XML-форматов.
- Каноническая сериализация даёт стабильный diff.
- Turtle является W3C-стандартом и поддерживается RDF-инструментами.

```bash
vedo-cli export --format turtle --ontology all --output ontology_dump.ttl
```

Protégé импортирует `.ttl` через File -> Open, начиная с версии 5.0.

## Экспорт ABox

ABox экспортируется в двух форматах: Turtle и JSON-LD.

| Формат | Преимущества | Когда использовать |
|--------|--------------|--------------------|
| Turtle | Компактный, стандартный, подходит для больших объёмов | Миграция в другую RDF-систему |
| JSON-LD | Удобен для web API, легко парсится в JavaScript | Интеграция с web-приложениями |

```bash
vedo-cli export --format turtle --type abox --output abox_data.ttl
vedo-cli export --format json-ld --type abox --output abox_data.jsonld
```

Восстановление ABox в новой системе:

- Turtle загружается через SPARQL INSERT или интерфейс triple store.
- JSON-LD загружается через HTTP API, например `POST /jsonld`.

## Экспорт Version Store

Version Store экспортируется как Git repository и JSON Lines logs.

Git repository является основным способом переноса истории.

```bash
vedo-cli export --format git --ontology main --output vedo_history.git
```

Если repository уже доступен как Git endpoint, допускается обычный clone:

```bash
git clone https://vedo.example/ontologies/main.git
```

JSON Lines используется для аналитики и инспекции без полного восстановления.

```bash
vedo-cli export --format jsonl --type commits --output commits.jsonl
```

Пример строки в `commits.jsonl`:

```json
{"hash":"abc123","author":"user@vedo.ai","date":"2025-01-15T10:30:00Z","message":"Added class Person","diff":{}}
```

## Экспорт audit logs

Audit logs экспортируются в JSON Lines (`.jsonl`), где одна строка соответствует одному событию. Формат подходит для потоковой загрузки в SIEM и аналитические системы.

```bash
vedo-cli export --format jsonl --type audit --start-date 2024-01-01 --end-date 2025-01-01 --output audit_export.jsonl
```

## Экспорт LFS-объектов

LFS-объекты экспортируются как raw files в исходном формате: PNG, PDF, ZIP и другие файлы не конвертируются.

```bash
mc sync minio/vedo-lfs/ ./vedo-lfs-export/
```

## Экспорт метрик и конфигурации

Конфигурация и RBAC экспортируются как YAML и JSON.

```bash
kubectl get configmap -n vedo -o yaml > vedo-config.yaml
kubectl get secret -n vedo -o yaml > vedo-secrets.yaml
vedo-cli export --format json --type access-control --output rbac.json
```

## Процедура вывода из эксплуатации

Единая процедура полного экспорта должна создавать структурированный каталог, включающий TBox, ABox, version history, audit logs, LFS, configuration, RBAC и manifest.

```bash
#!/bin/bash

OUTPUT_DIR="./vedo-decommission-$(date +%Y%m%d)"

mkdir -p "$OUTPUT_DIR"/{tbox,abox,version,audit,config,lfs}

vedo-cli export --format turtle --ontology all --output "$OUTPUT_DIR/tbox/ontology.ttl"

vedo-cli export --format turtle --type abox --limit 10000 --offset 0 --output "$OUTPUT_DIR/abox/abox_chunk_1.ttl"

vedo-cli export --format git --ontology main --output "$OUTPUT_DIR/version/main.git"

vedo-cli export --format jsonl --type audit --output "$OUTPUT_DIR/audit/audit.jsonl"

mc sync minio/vedo-lfs "$OUTPUT_DIR/lfs/"

kubectl get configmap,secret -n vedo -o yaml > "$OUTPUT_DIR/config/k8s-config.yaml"
vedo-cli export --format json --type rbac --output "$OUTPUT_DIR/config/rbac.json"
```

Каталог экспорта должен включать manifest как минимум со следующими данными:

- Версия VEDO Core.
- Дата и время экспорта.
- ID экспортированных онтологий.
- Список файлов и checksums.
- Тип данных для каждого файла.
- Формат и версия каждого экспортированного файла.
- Инструкции по импорту для нового экземпляра VEDO.
- Команды проверки целостности.

## Проверка целостности

| Тип данных | Инструмент проверки | Команда |
|------------|---------------------|---------|
| Turtle | RDF Validator | `rapper -c ontology.ttl` |
| Git | Git fsck | `git fsck --full` |
| JSON Lines | jq | `cat audit.jsonl | jq empty` |
| LFS | Сравнение хэшей | `md5sum -c checksums.txt` |

Пример проверки Turtle:

```bash
rapper -c ontology.ttl
```

Ожидаемый успешный результат: `rapper: Serializing Turtle succeeded`.

## Совместимость импорта

| Система | TBox Turtle | ABox Turtle | Version Git |
|---------|-------------|-------------|-------------|
| Protégé | Да | Да | Нет |
| TopBraid Composer | Да | Да | Нет |
| GraphDB | Да | Да | Нет |
| Apache Jena | Да | Да | Да, как обычный Git |
| RDF4J | Да | Да | Нет |

Git repository может быть импортирован только в другую систему, поддерживающую Git-историю VEDO или специальный скрипт восстановления metadata.

## Форматы экспорта для регуляторов

Если заказчику нужен специальный формат для регулятора, VEDO Core может предоставлять дополнительные конверсии.

| Формат | Инструмент | Применение |
|--------|------------|------------|
| XML / RDF-XML | `rdf-convert` | Старые системы |
| CSV | `vedo-cli export --format csv` | Аналитика, Excel |
| Parquet | `vedo-cli export --format parquet` | Data warehouse и big data |
| SQL INSERT | `vedo-cli export --format sql` | Ручное восстановление в реляционную БД |

```bash
vedo-cli export --format csv --type class --output classes.csv
vedo-cli export --format csv --type property --output properties.csv
vedo-cli export --format csv --type individual --output individuals.csv
```

## Статусная модель export package и обработка ошибок

### Базовый принцип

**Purge данных запрещён до successful integrity verification.** Экспортный пакет не считается валидным, пока все проверки не пройдены. Статус `failed_verification` блокирует любые destructive операции.

### Статусная модель

| Статус | Описание | Разрешённые действия |
|---|---|---|
| `in_progress` | Экспорт выполняется | Мониторинг, отмена |
| `verifying` | Экспорт завершён, проверяется целостность | Только read-only доступ к промежуточным файлам |
| `ready` | Все проверки пройдены, manifest подписан | Purge по запросу, скачивание, архивирование |
| `failed_verification` | Checksum mismatch, incomplete manifest или другая ошибка целостности | **Purge запрещён**, скачивание запрещено, только диагностика и перезапуск |
| `expired` | Старый пакет, вытеснен retention policy | Удаление самого пакета (не данных) |

### Сценарии ошибок

| Сценарий | Действие системы | Ожидаемое время реакции |
|---|---|---|
| **Failed export (процесс упал)** | Статус `failed_verification`, partial файлы помечены как invalid, администратор уведомлён | Автоматически, немедленно |
| **Checksum mismatch** | Статус `failed_verification`, mismatch логируется с указанием файла и expected/actual checksum, purge заблокирован | Автоматически при verify |
| **Incomplete manifest** | Статус `failed_verification`, перечень отсутствующих файлов логируется, purge заблокирован | Автоматически при verify |
| **Verification timeout** | Статус `failed_verification` с причиной `timeout`, администратор решает: перезапустить или увеличить timeout | По истечении настроенного лимита |

### Процедура восстановления после failed_verification

1. Администратор получает уведомление с причиной ошибки и списком затронутых артефактов
2. Partial файлы остаются доступны read-only для диагностики
3. Администратор выполняет `vedo-cli decommission --export --retry` (или `--clean` для полного перезапуска)
4. Новый экспорт проходит полный цикл: export → verify → status `ready` или `failed_verification`
5. Только после статуса `ready` администратор может выполнить `vedo-cli decommission --purge-data`

### Дополнительные инварианты

- Каждый export package имеет уникальный ID, timestamp и версию схемы manifest
- Manifest содержит полный перечень артефактов с checksums, форматами и retention policy
- Purge команда проверяет наличие валидного export package в статусе `ready`; при отсутствии — отказ с диагностикой
- Для air-gapped окружений verify выполняется локально, без внешних зависимостей
- Лог verification ошибок хранится вместе с manifest для аудита и поддержки

## Удаление данных после экспорта

После успешного экспорта (статус `ready`) и подтверждения заказчиком VEDO Core обязана удалить данные из системы, чтобы снизить риск утечки.

| Действие | Обязательность | Инструмент |
|----------|----------------|------------|
| Логическое удаление (soft-delete) | Обязательно | `vedo-cli decommission --purge-data` |
| Физическое удаление с диска | Опционально, требует доступа к инфраструктуре | `shred -vfz /var/lib/neo4j/data/databases/*` |
| Перезапись метаданных (wipe) | Для сверхчувствительных данных | `dd if=/dev/urandom of=/dev/sdaX` |

```bash
vedo-cli decommission --confirm --purge-data --export-first --export-dir ./vedo-decommission
```

Состав обязательного архивного пакета, сроки retention и правила передачи знаний определены в `decommission-archive-retention.md`.

Физическое уничтожение данных на носителях выполняется отдельно по согласованию и зависит от инфраструктуры заказчика. Подробная граница ответственности и процедуры зафиксированы в `secure-erase.md`.

## Формулировка для заказчика

При выводе VEDO Core из эксплуатации заказчик получает полный экспорт всех данных в открытых стандартных форматах:

- TBox: Turtle canonical, импортируется в Protégé, TopBraid, GraphDB.
- ABox: Turtle или JSON-LD.
- История коммитов: Git repository.
- Audit logs: JSON Lines.
- Бинарные файлы: исходный формат.
- Конфигурация и RBAC: YAML/JSON.

Экспорт включает manifest, checksums, integrity verification script и import instructions.

По запросу данные могут быть дополнительно конвертированы в CSV, Parquet, SQL или RDF/XML.

Данные удаляются из системы после подтверждения заказчиком через logical purge. Физическое уничтожение данных на носителях выполняется отдельно по согласованию.

## Бизнес-правила

- Экспорт при выводе из эксплуатации должен использовать открытые, непроприетарные форматы.
- TBox должен экспортироваться как canonical Turtle.
- ABox должен экспортироваться как Turtle и JSON-LD.
- Version Store должен экспортироваться как Git repository и JSON Lines commit logs.
- Audit logs должны экспортироваться как JSON Lines / NDJSON.
- LFS objects должны экспортироваться как raw files без конвертации.
- Configuration и RBAC должны экспортироваться как YAML и JSON.
- Export package должен включать manifest, checksums и инструкции по integrity verification.
- Экспорт ABox должен поддерживать chunking для больших наборов данных.
- Customer-specific regulatory export formats могут включать RDF/XML, CSV, Parquet и SQL INSERT.
- Удаление данных после экспорта требует подтверждения заказчика.
- Logical purge после подтверждённого экспорта обязателен.
- Physical destruction является опциональным и требует отдельного соглашения и доступа к инфраструктуре.
- Ответственность за secure erase и требования к evidence определены в `secure-erase.md`.

## Versioned Export Format (дополнение)

### Принцип

Каждый экспортный пакет должен содержать версию формата данных, чтобы обеспечить читаемость архивов через 5+ лет.

**Поле в манифесте:**
```yaml
# manifest.yaml
export_format_version: 1
export_timestamp: 2026-05-23T10:00:00Z
tool_version: "vedo-cli/1.2.3"
```

### Политика версионирования

| Версия | Формат | Когда введена | Статус | Конвертер в версию N+1 |
|--------|--------|----------------|--------|------------------------|
| **1** | Canonical Turtle (`.ttl`), JSON-LD (`.jsonld`), Git bundle | MVP (2026) | Активна | `vedo-cli migrate-archive --from-version 1 --to-version 2` |
| **2** | (резерв) | TBD | — | — |

### Требования к долговременной читаемости

| Требование | Реализация |
|------------|------------|
| Формат должен быть документирован | Спецификация в `docs/export-format-v1.md` |
| Должен существовать конвертер в актуальную версию | `vedo-cli archive-convert --from-version 1` |
| Конвертер должен работать без доступа к VEDO Core | Только стандартные библиотеки (Rust, Python, Go) |
| Тест на читаемость | CI ежегодно проверяет чтение архивов v1 и преобразование в v2 |

### Процедура миграции формата архивов

1. **Создание новой версии формата (v2):**
   - Документирование изменений в `docs/export-format-v2.md`.
   - Реализация конвертера `v1 → v2` (в `vedo-cli archive-convert`).

2. **Обновление существующих архивов:**
   - `vedo-cli archive-migrate --bucket s3://vedo-archives/ --target-version 2`
   - Job выполняется асинхронно (может занимать дни для больших архивов).
   - Оригинальные архивы v1 сохраняются (WORM) как резервная копия.

3. **Периодичность:**
   - При каждом изменении формата (мажорная версия VEDO Core).
   - Но не реже 1 раза в 2 года (даже без изменений — проверка конвертера).

### Ответственность

| Роль | Ответственность |
|------|-----------------|
| **Architect** | Определение новой версии формата (при breaking change) |
| **DevOps** | Запуск `archive-migrate` после обновления |
| **Security Lead** | Проверка, что конвертер не раскрывает данные |

## Открытые вопросы

- Нет открытых вопросов по D1.1.
