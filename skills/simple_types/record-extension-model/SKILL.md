---
description: Стандарты расширенной модели записи (REM) — модели, коллекции, атрибуты, choices, клиентские скрипты, элементы формы. Загружать при работе с rem/**.
paths: '**/rem/**/*'
user-invocable: false
---

# Расширенная модель записи (REM)

DSL-экспорт: TypeScript для метаданных, JavaScript для скриптов. Классы: `SysReModel`, `SysReCollection`, `SysReAttribute`, `SysReChoice`, `SysReModelClientScript`, `SysReFormElement`. `sys_id` — только в экспорте, при создании убрать. Описаны только значимые поля; остальные — дефолты.

Docs: [REM](https://docs.simpleone.ru/ru/platform/admin-help/data-structure/tables/record-extended-model) · [Extend data models](https://docs.simpleone.ru/ru/platform/developer-help/data-structure/extend-data-models/) · [Use cases](https://docs.simpleone.ru/ru/platform/developer-help/data-structure/extend-data-models/rem-use-cases). Скрипты — `script-rules`.

## Концепция

REM расширяет набор полей таблицы без дочерних таблиц/колонок — вместо колонок **атрибуты**, выборочно применяемые к записям.
- **Модель** (`SysReModel`) — набор атрибутов для конкретной таблицы; к записи привязывается одна модель.
- **Коллекция** (`SysReCollection`) — переиспользуемый набор атрибутов, связывается с несколькими моделями (M2M). Рекомендуется по одному атрибуту на коллекцию для гибкости.
- **Атрибут** (`SysReAttribute`) — аналог колонки. Все типы, **кроме** Date/Time Specific, Record Class, Journal Input, Document ID.
- **Контейнер** (`sys_re_container`) — базовая таблица для моделей и коллекций; обеспечивает полиморфную ссылку `re_container_id`. Напрямую не создаётся.

**Ограничения связей**: коллекция с `table_id` — только к моделям той же/дочерней таблицы; модель и коллекция с пересекающимися именами атрибутов связать нельзя; связь модель↔коллекция — только один раз.

## Порядок импорта (⚠️ нарушение → ошибка «должно быть Integer»)

`SysReCollection`(SysDbTable опц.) → `SysReModel`(SysDbTable) → `SysReAttribute`(re_container_id) → `SysReChoice`(column_id→attribute) → `SysReModelClientScript`(re_container_id, re_attribute_id для onChange) → `SysReFormElement`(re_container_id, re_attribute_id).

## Уникальность

| Сущность | Ключ |
|----------|------|
| `sys_re_attribute` | `column_name` + `re_container_id` (✅) |
| `sys_re_choice` | `column_id` + `value` (✅) |
| `sys_re_form_element` | `re_attribute_id` + `re_container_id` (✅, атрибут на форме один раз) |
| `sys_re_model` / `sys_re_collection` / `sys_re_model_client_script` | `title`+`table_id` / `title` / `name`+`re_container_id` (желательно) |

## Размещение и ключевые поля

Имя константы: `toCamelCase(display_name||name||title, 50)`, fallback `<type>_${sys_id}`. Reference-поля резолвятся в импорты автоматически; файлы скриптов импортируются с `as unknown as string`.

| Тип | Путь | Ключевые поля |
|-----|------|---------------|
| `SysReModel` | `rem/models/{table.name}/records.ts` | `title`✱, `table_id`✱ (после сохранения read-only), `after_insert_script` (серверный контекст), `active`, `icon` |
| `SysReCollection` | `rem/collections.ts` | `title`✱, `active`, `table_id` (ограничение по таблице) |
| `SysReAttribute` | `rem/attributes/{container.name‖kebab(title)}.ts` | `re_container_id`✱, `column_type_id`✱ (имя типа), `title`✱, `column_name`✱ (авто из title, read-only после сохранения), `map_to_column_id` (связь с колонкой целевой таблицы, типы должны совпадать), `mandatory`, `read_only`, `full_text_search`, `default_value` |
| `SysReChoice` | `rem/choices/{attribute.name‖kebab(title)}.ts` | `column_id`✱ (→attribute), `value`✱ (database, `lowercase_snake_case` латиница — см. `data-structure`), `title`✱ (display), `order` |
| `SysReModelClientScript` | `rem/client-scripts/{container.name‖kebab(title)}/records.ts` (+ `.js` рядом) | `name`✱, `re_container_id`✱, `type`✱, `re_attribute_id` (обязателен для `onChange`, иначе null), `script`✱, `active` |
| `SysReFormElement` | `rem/form-elements/{container.name‖kebab(title)}.ts` | `re_container_id`✱, `re_attribute_id`✱, `position` |

Тип клиентского скрипта: `onLoad` (re_attribute_id null), `onChange` (✅ ссылка на атрибут), `onSubmit` (null; `return false` отменяет отправку). Контекст — **клиентский** (`script-rules`). Под `<remform>` `s_form` берётся из контекста remform; REM-методы: `s_form.getREMValue/setREMValue/setREMMandatory/setREMReadOnly/setREMVisible/getREMDisplayValue/getREMLabelOf/setREMLabelOf`.

## Примеры DSL

```typescript
// rem/models/task/records.ts
import { SysReModel } from "../../../../../platform/data-structure/columns/sys_re_model";
import { task } from "../../../../../sa/domain/data-structure/tables/task";
export const equipmentRequest: SysReModel = new SysReModel({
    "title": "Запрос на оборудование", "table_id": task, "active": true,
});

// rem/collections.ts
export const delivery: SysReCollection = new SysReCollection({ "title": "Доставка", "active": true, "table_id": task });

// rem/attributes/equipment-request.ts
export const equipmentType: SysReAttribute = new SysReAttribute({
    "re_container_id": equipmentRequest, "column_type_id": "Choice", "title": "Тип оборудования",
    "column_name": "equipment_type", "mandatory": true, "active": true,
});

// rem/choices/equipment-type.ts
export const laptop: SysReChoice = new SysReChoice({
    "column_id": equipmentType, "value": "laptop", "title": "Ноутбук", "order": 100,
});

// rem/client-scripts/task-type/records.ts (+ change-model.js)
import changeModelScript from "./change-model.js";
export const changeModel: SysReModelClientScript = new SysReModelClientScript({
    "name": "Изменить модель", "re_container_id": taskType, "type": "onChange",
    "re_attribute_id": relatedModel, "active": true, "script": changeModelScript as unknown as string,
});

// rem/form-elements/equipment-request.ts
export const equipmentTypeElement: SysReFormElement = new SysReFormElement({
    "re_container_id": equipmentRequest, "re_attribute_id": equipmentType, "position": 1,
});
```
```javascript
// change-model.js — onChange: эмитит событие для отрисовки атрибутов выбранной модели
export default function script() {
  (function() { SimpleEventBus.emit('modelChanged', s_form.getValue('related_model')); })();
}
```

## Отображение на формах

SimpleTags `<rem>` (секция в существующей форме) и `<remform>` (самостоятельная форма) внутри виджетов. Атрибуты — см. `design-system-rules` §5: оба требуют `tableName`✱, опционально `modelId`, `recordId`, `isPortal`, `isUserScripts`, `style`; `<remform>` доп.: `name`, `saveButtonCaption`, `hideSaveButton`. Виджет создаётся по `widgets`, экземпляр встраивается через `SysUiFormElement.widget_instance_id`.

## Серверный доступ (`rem_attr`)

```javascript
const record = new SimpleRecord('task');
record.get(recordId);
if (record.getReModelId()) {                                   // проверка наличия модели
    const value = record.rem_attr.getValue('equipment_type');
    record.rem_attr.setValue('inventory_number', 'INV-001');
    record.rem_attr.getDisplayValue('equipment_type');         // также getTitle(name)
}
```
