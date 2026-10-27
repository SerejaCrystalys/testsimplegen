---
description: Стандарты форм — Form View, секции, элементы, Related Lists. Загружать при работе с forms/**.
paths: '**/forms/**/*'
user-invocable: false
---

# Формы (Views, Sections, Elements, Related Lists)

DSL-экспорт, TypeScript. Классы: `SysUiForm`, `SysUiFormSection`, `SysUiFormElement`, `SysUiRelatedList`, `SysUiRelatedListElement`. `sys_id` — только в экспорте. [Docs](https://docs.simpleone.ru/platform/admin-help/forms/).

## Иерархия и порядок генерации (⚠️ КРИТИЧНО)

Каждая сущность зависит от предыдущей — нельзя создать секцию без формы, элемент без секции, связанный список без формы:

```
SysUiForm → SysUiFormSection (form_id) → SysUiFormElement (form_section_id)
          → SysUiRelatedList (form_id) → SysUiRelatedListElement (related_list_id)
```

Уникальность: form — `title`+`table_id`; element — `column_id`+`form_section_id`; related_list — `table_id`+`form_id`+`title`; related_list_element — `related_column_id`+`related_list_id`.

## Размещение и поля

Все — в `forms/{form.table_id.name}/`, группировка по таблице формы. Константа — `toCamelCase(display_name||name||title, 50)`.

| Тип | Файл | Ключевые поля |
|-----|------|---------------|
| `SysUiForm` | `views.ts` | `table_id`✱, `title` (стандарт — `"Default"`) |
| `SysUiFormSection` | `sections.ts` | `form_id`✱, `title` (Translated Text — через `sys_translation`, **не** SourceMessage), `position` (с 1) |
| `SysUiFormElement` | `elements.ts` | `form_section_id`✱, `position` (с 1), `column_id`, `widget_instance_id`, `form_split_element_type` |
| `SysUiRelatedList` | `related-lists.ts` | `form_id`✱, `table_id`✱ (связанная таблица), `title` |
| `SysUiRelatedListElement` | `related-list-elements.ts` | `related_list_id`✱, `related_column_id`✱, `title`, `order` (с 1) |

**Типы Form Element**: колонка таблицы — `column_id`=ссылка, `widget_instance_id`/`form_split_element_type`=null; виджет — `column_id`=null, `widget_instance_id`=ссылка на `SysWidgetInstance` (порядок `SysWidget`→`SysWidgetInstance`→`SysUiFormElement`, см. `widgets`).

```typescript
// views.ts
export const default_: SysUiForm = new SysUiForm({ "table_id": pdaProject, "title": "Default" });
// sections.ts
export const general: SysUiFormSection = new SysUiFormSection({ "title": "General", "form_id": default_, "position": 1 });
// elements.ts — колонка и виджет
export const nameElement: SysUiFormElement = new SysUiFormElement({
    "form_section_id": general, "column_id": pdaName, "form_split_element_type": null, "position": 1,
});
export const summaryWidget: SysUiFormElement = new SysUiFormElement({
    "form_section_id": general, "widget_instance_id": projectSummaryInstance, "column_id": null, "position": 2,
});
// related-lists.ts + related-list-elements.ts
export const tasks: SysUiRelatedList = new SysUiRelatedList({ "table_id": pdaTask, "form_id": default_, "title": "Tasks" });
export const taskName: SysUiRelatedListElement = new SysUiRelatedListElement({
    "related_list_id": tasks, "related_column_id": pdaName, "title": "Name", "order": 1,
});
```
