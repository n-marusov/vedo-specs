# ADR-DES.SECURITY.gitlab-like-organization-model

**Дата:** 2026-05-13  
**Статус:** Принято

## Контекст

VEDO Core должен поддерживать многокомандную и многопроектную работу. Пользователям нужна понятная модель групп, онтологий, ролей и наследования доступа, близкая к привычному рабочему процессу GitLab.

Ранее в артефактах использовались разные формулировки: `проект`, `онтология`, `администратор`, `Maintainer`. Для консистентности требуется зафиксировать единую модель.

## Требование-источник
- [organization-access-model.md](requirements/REQ-NFR.SECURITY.organization-access-model.md)

## Решение

Использовать GitLab-like модель организации: Group как контейнер команд, Project как единица доступа и версионирования; Ontology — содержимое Project (TBox/ABox, классы, свойства, индивиды, аксиомы), роли Viewer/Editor/Maintainer/Owner с наследованием по иерархии и атрибутными правами через семантические паттерны графа.

Project и Ontology связаны 1:1 — один Project содержит ровно одну Ontology. Группировка нескольких онтологий выполняется через Group, а не через упаковку в один Project. Membership, visibility и ABAC policies хранятся на Project; Ontology наследует их через 1:1-связь.

Модель понятна пользователям, знакомым с GitLab, и даёт явную границу ответственности: Membership boundary — только Owner, Maintainer сфокусирован на качестве изменений через Ontology Merge Request и protected main. Атрибутные права через graph patterns (URI prefix, parent class, relationship) вместо строковых масок обеспечивают гранулярный контроль без разрыва графовой модели онтологии.

Owner управляет членством и ролями; Maintainer — рабочий процесс и OMR review/merge; атрибутные права задаются семантическими паттернами с правилом most specific wins; write_with_approval реализуется через Ontology Merge Request с proposal branch и semantic diff.

## Рассмотренные альтернативы

| Альтернатива | Причина отклонения |
|--------------|--------------------|
| Только роли Viewer/Editor/Maintainer без Owner | Не отделяет управление членством от управления рабочим процессом онтологии |
| Project содержит несколько онтологий | Противоречит GitLab-модели «один репозиторий = один Project» и нарушает 1:1 Project ↔ Ontology; группировка нескольких онтологий выполняется через Group |
| Maintainer управляет членством | Смешивает review/merge responsibilities с ownership и создаёт риск несанкционированной раздачи прав |
| Строковые маски для атрибутных прав | Плохо подходят к графовой модели онтологии; нужны семантические паттерны |

## Последствия

**Положительные последствия:**
- Модель понятна пользователям, знакомым с GitLab.
- Membership boundary становится явным: только Owner управляет участниками.
- Maintainer сфокусирован на качестве изменений и protected `main`.
- Ontology становится понятной единицей доступа, истории и владения.

**Отрицательные последствия:**
- Появляется дополнительная роль Owner и необходимость UI для управления ownership.
- Атрибутные политики требуют визуального конструктора и безопасного исполнения graph pattern.

**Меры снижения рисков:**
- Все membership operations требуют Owner check и audit log.
- Maintainer actions ограничиваются настройками рабочего процесса и OMR review/merge.
- Attribute policy execution использует allowlisted graph patterns, а не произвольный SPARQL.

## Связанные ADR

- [ADR-DES.API.organization-rest-endpoints.md](ADR-DES.API.organization-rest-endpoints.md) — канонический REST-контракт для organization model (`/groups/:id/...`, `/projects/:id/...`), включая members/visibility/policies endpoints на Project.

---
