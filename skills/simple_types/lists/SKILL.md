---
description: Стандарты списковых представлений (List Views) и их колонок. Загружать при работе с lists/**.
paths: '**/lists/**/*'
user-invocable: false
---

# Списки (List Views)

DSL-экспорт, TypeScript. Классы: `SysUiList`, `SysUiListColumn`. `sys_id` — только в экспорте. [Docs](https://docs.simpleone.ru/platform/admin-help/lists/). Related Lists — в `forms` (относятся к форме).

Уникальность: list — `title`+`table_id`; column — `list_id`+`column_id`. Размещение в `lists/{table.name}/`, группировка по таблице. Константа — `toCamelCase(...)`.

- **`SysUiList`** (`views.ts`): `table_id`✱, `title` (стандарт — `"Default"`).
- **`SysUiListColumn`** (`columns.ts`): `list_id`✱, `column_id`✱ (колонка должна принадлежать `table_id` списка или родителю), `order` (слева направо, **с 0**).

Порядок: сначала `SysUiList`, затем `SysUiListColumn` — без колонок список пуст.

```typescript
// views.ts
export const default_: SysUiList = new SysUiList({ "table_id": pdaProject, "title": "Default" });
// columns.ts
import { pdaName, pdaState, pdaOwner } from "../../data-structure/columns/pda_project";
export const colName: SysUiListColumn = new SysUiListColumn({ "list_id": default_, "column_id": pdaName, "order": 0 });
export const colState: SysUiListColumn = new SysUiListColumn({ "list_id": default_, "column_id": pdaState, "order": 1 });
export const colOwner: SysUiListColumn = new SysUiListColumn({ "list_id": default_, "column_id": pdaOwner, "order": 2 });
```
