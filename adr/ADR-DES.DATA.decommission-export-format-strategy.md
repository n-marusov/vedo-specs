# ADR-DES.DATA.decommission-export-format-strategy

**Дата:** 2026-05-10  
**Статус:** Принято

## Контекст

VEDO Core должен поддерживать контролируемый вывод системы из эксплуатации без привязки к поставщику и без потери смысла данных. Заказчик должен получить данные в открытых форматах, пригодных для инспекции, архивирования, регуляторной передачи и импорта в другие RDF/ontology-системы.

## Требование-источник
- [decommission-export.md](requirements/REQ-NFR.DATA.decommission-export.md)

## Решение

Принять стратегию экспорта в открытых форматах (Turtle, JSON-LD, JSON Lines, YAML) при выводе системы из эксплуатации без привязки к поставщику.

Открытые форматы гарантируют переносимость данных в Protégé, TopBraid, GraphDB, Apache Jena и другие RDF-инструменты. Manifest и checksums позволяют доказать полноту и целостность выгрузки. Статусная модель пакета (`in_progress` → `verifying` → `ready` → `failed_verification` → `expired`) предотвращает преждевременное удаление данных при повреждении экспорта.

TBox экспортировать в canonical Turtle, ABox в Turtle + JSON-LD, Version Store в Git + JSON Lines, audit logs в JSON Lines, LFS-объекты без изменения формата, конфигурацию и RBAC в YAML + JSON. Purge данных разрешить только после успешной проверки целостности и подтверждения заказчиком.

## Рассмотренные альтернативы

| Альтернатива | Причина отклонения |
|--------------|--------------------|
| Проприетарный VEDO-only dump | Создаёт привязку к поставщику и не гарантирует переносимость в другие RDF-системы |
| Только RDF/XML | Менее человекочитаем, хуже diff, больше XML-шума |
| Только Turtle для всех данных | Не подходит для version history, audit logs, configuration и binary objects |
| SQL dump как основной формат | Завязан на внутреннюю схему хранения и не переносит семантику ontology-first модели |
| Удалять данные без manifest и проверки checksums | Повышает риск необнаруженной потери или повреждения данных |

## Последствия

**Положительные последствия:**
- Заказчик получает переносимые открытые форматы без привязки к VEDO.
- TBox и ABox можно импортировать в распространённые RDF/ontology tools.
- Version history сохраняется в Git-compatible виде.
- Audit logs подходят для SIEM и long-term archive.
- Manifest и checksums позволяют доказать полноту и целостность выгрузки.

**Отрицательные последствия:**
- Полный конвейер экспорта сложнее, чем единый database dump.
- Для больших ABox требуется chunking и отдельная проверка целостности.
- Git export может быть полезен только системам, которые понимают VEDO version semantics или имеют conversion scripts.
- Physical destruction не может быть универсально гарантирован без доступа к инфраструктуре заказчика.

**Меры снижения рисков:**
- Поставлять `vedo-cli export` и decommission runbook.
- Включать manifest, checksums и verification commands в каждый export package.
- Использовать canonical Turtle для стабильного diff и reproducible export.
- Документировать import paths для Protégé, TopBraid, GraphDB, Apache Jena и VEDO.
- Отделять mandatory logical purge от optional physical destruction по согласованию.

---
