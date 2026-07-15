# Схема sequence операций для извлечения онтологии

| Атрибут | Значение |
|---------|----------|
| **ID** | REQ-FUN.API.doc-extract-sequence-schema |
| **Уровень** | FUN |
| **Атрибут качества** | Functionality |
| **Приоритет** | P0 |
| **Статус** | ЧЕРНОВИК |
| **Источник** | vision.md (F14.6) |

---

## Назначение

Определить JSON-схему sequence операций, которую LLM генерирует при извлечении онтологии из документа.

## Требование

LLM должна возвращать sequence операций в строгом JSON-формате для последующего применения через gRPC ApplySequence.

### JSON Schema

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "title": "OntologyBuildSequence",
  "type": "object",
  "properties": {
    "ontology_name": { "type": "string" },
    "steps": {
      "type": "array",
      "items": { "$ref": "#/definitions/Step" }
    }
  },
  "required": ["steps"],
  "definitions": {
    "Step": {
      "type": "object",
      "properties": {
        "op": {
          "type": "string",
          "enum": [
            "create_class",
            "set_parent",
            "add_parent",
            "create_object_property",
            "create_datatype_property",
            "set_domain",
            "set_range",
            "add_annotation",
            "create_individual",
            "set_property_value",
            "add_equivalence",
            "add_disjoint"
          ]
        },
        "id": { "type": "string", "pattern": "^[A-Za-z][A-Za-z0-9_-]*$" },
        "label": { "type": "string" },
        "comment": { "type": "string" },
        "parent": { "type": "string" },
        "domain": { "type": "string" },
        "range": { "type": "string" },
        "data_type": {
          "type": "string",
          "enum": ["string", "integer", "decimal", "boolean", "date", "dateTime", "float", "double"]
        },
        "characteristics": {
          "type": "array",
          "items": {
            "type": "string",
            "enum": ["transitive", "symmetric", "asymmetric", "functional", "inverse-functional", "reflexive", "irreflexive"]
          }
        },
        "annotations": {
          "type": "object",
          "additionalProperties": { "type": "string" }
        },
        "value": { "type": "string" }
      },
      "required": ["op", "id"],
      "allOf": [
        {
          "if": { "properties": { "op": { "enum": ["create_class", "set_parent", "add_parent"] } } },
          "then": { "required": ["id"] }
        },
        {
          "if": { "properties": { "op": { "enum": ["create_object_property", "create_datatype_property"] } } },
          "then": { "required": ["id", "domain", "range"] }
        },
        {
          "if": { "properties": { "op": { "enum": ["set_domain"] } } },
          "then": { "required": ["id", "domain"] }
        },
        {
          "if": { "properties": { "op": { "enum": ["set_range"] } } },
          "then": { "required": ["id", "range"] }
        },
        {
          "if": { "properties": { "op": { "enum": ["create_individual"] } } },
          "then": { "required": ["id", "parent"] }
        },
        {
          "if": { "properties": { "op": { "enum": ["add_annotation"] } } },
          "then": { "required": ["id", "range", "value"] }
        }
      ]
    }
  }
}
```

### Пример

```json
{
  "ontology_name": "ecommerce",
  "steps": [
    { "op": "create_class", "id": "Customer", "label": "Клиент", "comment": "Зарегистрированный пользователь магазина" },
    { "op": "create_class", "id": "Product", "label": "Товар" },
    { "op": "set_parent", "id": "Customer", "parent": "owl:Thing" },
    { "op": "create_object_property", "id": "purchases", "domain": "Customer", "range": "Product", "characteristics": ["transitive"] },
    { "op": "create_datatype_property", "id": "email", "domain": "Customer", "range": "string" },
    { "op": "add_annotation", "id": "Customer", "range": "rdfs:comment", "value": "Основной субъект电商 системы" }
  ]
}
```

## Критерии приёмки

1. LLM возвращает sequence в строгом соответствии с JSON Schema.
2. Все id уникальны в рамках одного sequence.
3. Ссылочная целостность: все domain/range/parent ссылаются на id, объявленные ранее в sequence.
4. Нет циклических иерархий (A → B → A).
5. Schema валидируется до отправки sequence в gRPC.
