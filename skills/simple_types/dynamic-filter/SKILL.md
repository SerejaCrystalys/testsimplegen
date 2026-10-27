---
description: Стандарты динамических фильтров (Filter Option Dynamics). Загружать при работе с data-structure/dynamic-filter-options/**.
paths: '**/data-structure/dynamic-filter-options/**/*'
user-invocable: false
---

# Динамические фильтры (Dynamic Filter Options)

DSL-экспорт, TypeScript. Класс `SysFilterOptionDynamic`. `sys_id` — только в экспорте. [Docs](https://docs.simpleone.ru/platform/admin-help/auxiliary-elements/condition-builder/filters#dynamic-filters).

Уникальность: `title` + `referenced_table_id`. Размещение: `data-structure/dynamic-filter-options/{referenced_table.name}/records.ts`, скрипты фильтров — отдельными `.js` рядом; группировка по `referenced_table_id`. Константа — `toCamelCase(title, 50)`.

## Связь с Reference-колонками

Используются в `SysDbColumn` через `dynamic_reference_qualifier_id`. **Должны быть созданы до** колонки, которая на них ссылается (иначе ошибка `"dynamic_reference_qualifier_id должно быть Integer"`). См. `data-structure`.

## Скрипт (серверный контекст)

Доступно: `ss` (`getUserId`, `getSession`, `getProperty`), `current_user` (текущий пользователь), `SimpleRecord`. Скрипт возвращает Encoded Query или значение. **Запрещено** (ServiceNow/Client API): `getRefRecord()` → `new SimpleRecord(table).get(id)`; `current.getEncodedQuery()` → формировать строкой; `s_form.*`/`s_list.*` → `ss.*`/`SimpleRecord`.

```typescript
import { SysFilterOptionDynamic } from "../../../../platform/data-structure/columns/sys_filter_option_dynamic";
import { sysUser } from "../tables/sys_user";

export const me: SysFilterOptionDynamic = new SysFilterOptionDynamic({
    "title": "Me", "referenced_table_id": sysUser,
    "script": "return current_user.getValue('sys_id');",
});

// Колонки выбранной в форме таблицы (+ фильтр по типу через ^column_typeIN...)
export const numericColumnsOfTable: SysFilterOptionDynamic = new SysFilterOptionDynamic({
    "title": "Numeric columns of selected table", "referenced_table_id": sysDbColumn,
    "script": "const tableId = ss.getSession().getParameter('table_id'); return 'table_id=' + tableId + '^column_typeINInteger,Decimal,Float';",
});
```

**Условные квалифаеры** (зависимость от нескольких полей): создать отдельный `SysFilterOptionDynamic` на каждый вариант, а клиентский скрипт `onChange` переключает `dynamic_reference_qualifier_id` у Reference-колонки при изменении зависимого поля.
