---
description: Стандарты навигационного меню — категории и пункты. Загружать при работе с navigation/**.
paths: '**/navigation/**/*'
user-invocable: false
---

# Навигационное меню (Categories, Items)

DSL-экспорт, TypeScript. Классы: `SysMenuCategory`, `SysMenuItem`. `sys_id` — только в экспорте. [Docs](https://docs.simpleone.ru/platform/admin-help/navigation/).

## Иерархия и порядок

```
Menu Category (parent_id — самореференс для вложенности)
  └── Menu Item (category_id → Menu Category)
```

Категории создаются **до** пунктов. Родительская категория в файле — **выше** дочерней (самореференс `parent_id`). Уникальность (желательно): category `title`; item `title`+`category_id`. Навигационные сущности не содержат скриптов/HTML. Все категории — в `navigation/menu-categories.ts`, все пункты — в `navigation/menu-items.ts`. Константа — `toCamelCase(...)`.

## Поля

- **`SysMenuCategory`**: `title` (Translated Text — через `sys_translation`, **не** SourceMessage), `parent_id` (категория или null), `order` (шаг 100), `active`, `policy`.
- **`SysMenuItem`**: `title` (Translated Text), `category_id`✱ (⚠️ имя поля — `category_id`, не `menu_category_id`), `table_id` (список таблицы) **или** `url` (произвольная страница — обычно одно из двух), `link_type` (`"list"`/`"url"`), `order`, `filter` (Encoded Query фильтр списка или null), `user_roles` (массив ролей для ограничения видимости или null), `essence_id`, `active`, `policy`.

`filter` примеры: `null` (все), `"(demo_stage=done)"` (статический), `"(demo_assigneeDYNAMIC156957117519820256)"` (динамический — `columnDYNAMICsys_id`, где sys_id — ID `SysFilterOptionDynamic`; для нового использовать ID платформенного фильтра или создать свой).

```typescript
// menu-categories.ts
export const incidents: SysMenuCategory = new SysMenuCategory({ "title": "Incidents", "parent_id": null, "order": 100, "active": true, "policy": "Open" });
export const reports: SysMenuCategory = new SysMenuCategory({ "title": "Reports", "parent_id": incidents, "order": 100, "active": true, "policy": "Open" });

// menu-items.ts
import { incidents } from "./menu-categories";
import { demoStory } from "../data-structure/tables/demo_story";
import { demoAdmin } from "../access-control/roles";
export const myStories: SysMenuItem = new SysMenuItem({
    "title": "My Stories", "category_id": incidents, "link_type": "list", "table_id": demoStory, "url": "",
    "filter": "(demo_assigneeDYNAMIC156957117519820256)", "user_roles": null, "order": 100,
    "active": true, "policy": "Open", "essence_id": null,
});
export const releasedStories: SysMenuItem = new SysMenuItem({
    "title": "Released Stories", "category_id": incidents, "link_type": "list", "table_id": demoStory, "url": "",
    "filter": "(demo_stage=done)", "user_roles": [demoAdmin], "order": 500,
    "active": true, "policy": "Open", "essence_id": null,
});
```

Практики: группировать через `parent_id` (не плоские структуры); шаг `order` 100; `active: false` вместо удаления.
