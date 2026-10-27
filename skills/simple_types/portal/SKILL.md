---
description: Стандарты портальных сущностей — Page, Container, Row, Column, Portal, Context, Node. Загружать при работе с portal/** (плюс widgets для инстансов виджетов).
paths: '**/portal/**/*'
user-invocable: false
---

# Портальные сущности

DSL-экспорт, TypeScript. Классы: `Page`, `PageContainer`, `PageRow`, `PageColumn`, `Portal`, `SysPortalContext`, `PortalNode`. `sys_id` — только в экспорте. Каждый тип — в отдельном файле `portal/<name>.ts`. Константа — `toCamelCase(...)`. [Docs](https://docs.simpleone.ru/ru/platform/admin-help/portals/portal-pages).

## Иерархия и порядок импорта (⚠️ КРИТИЧНО)

```
Page → PageContainer (page_id) → PageRow (page_container_id) → PageColumn (page_row_id) → WidgetInstance
Portal (page_*_id, *_container_id) → SysPortalContext / PortalNode (portal_id)
```

Порядок: `pages` → `page-containers` → `page-rows` → `page-columns` → `portals` → `portal-contexts`/`portal-nodes`. Уникальность: `page.path_name` и `portal.path_name` — глобально.

## Поля

| Тип / файл | Ключевые поля |
|------------|---------------|
| `Page` / `pages.ts` | `path_name`✱ (URL-суффикс, глобально уникален), `title`✱ (Translated Text), `description`, `active`, `policy` |
| `PageContainer` / `page-containers.ts` | `page_id`✱, `name`, `sys_order`✱ (шаг 100), `active`, `policy` |
| `PageRow` / `page-rows.ts` | `page_container_id`✱, `sys_order`✱, `active`, `policy` |
| `PageColumn` / `page-columns.ts` | `page_row_id`✱, `sys_order`✱, `size_xs`/`size_sm`/`size_md`/`size_lg`✱ (12-колоночная сетка; сумма колонок строки на брейкпоинт = 12), `active`, `policy` |
| `Portal` / `portals.ts` | `path_name`✱, `title`✱, `page_home_id`✱, `page_login_id`/`page_404_id` (или null), `header_container_id`/`footer_container_id` (ссылки на **PageContainer**, не Page; или null), `active`, `policy` |
| `SysPortalContext` / `portal-contexts.ts` | `portal_id`✱, `user_criteria_ids`✱ (**массив** `SysUserCriteria[]`, тип List), `active`, `policy` |
| `PortalNode` / `portal-nodes.ts` | `portal_id`✱, `path_name`✱, `page_id`✱, `parent_node_id`, + catalog-поля (см. ниже) |

**PortalNode — два режима**: узел-ссылка (навигация) — заполнить `portal_id`/`path_name`/`page_id`, все catalog-поля null. Узел-каталог — заполнить все: `page_id_for_item`, `item_table_id`, `item_parent_category_column_id`, `category_table_id`, `category_parent_column_id`. (В DSL parent-ссылка не зарегистрирована — иерархия по `order`.)

**Виджеты**: `SysWidgetInstance.page_column_id` ссылается на `PageColumn` для портальных виджетов (для виджетов на формах — `null`, встраивание через `SysUiFormElement`, см. `widgets`).

Translated Text (через `sys_translation`, не SourceMessage): `Page.title`/`description`, `PageContainer.name`, `Portal.title`.

## Примеры

```typescript
// pages.ts
export const homePage: Page = new Page({ "path_name": "home", "title": "Home Page", "active": true, "policy": "Open" });
// page-containers.ts → page-rows.ts → page-columns.ts
export const homeContainer: PageContainer = new PageContainer({ "page_id": homePage, "name": "Main", "sys_order": 100, "active": true, "policy": "Open" });
export const homeRow: PageRow = new PageRow({ "page_container_id": homeContainer, "sys_order": 100, "active": true, "policy": "Open" });
export const homeColLeft: PageColumn = new PageColumn({
    "page_row_id": homeRow, "sys_order": 100, "size_xs": 12, "size_sm": 12, "size_md": 8, "size_lg": 8, "active": true, "policy": "Open",
});
// portals.ts
export const servicePortal: Portal = new Portal({
    "path_name": "portal", "title": "Service Portal", "page_home_id": homePage,
    "page_login_id": loginPage, "page_404_id": notFoundPage,
    "header_container_id": headerContainer, "footer_container_id": footerContainer, "active": true, "policy": "Open",
});
// portal-contexts.ts (user_criteria_ids — массив)
export const agentContext: SysPortalContext = new SysPortalContext({
    "portal_id": servicePortal, "user_criteria_ids": [itsmAgentCriteria], "active": true, "policy": "Open",
});
// portal-nodes.ts — узел-каталог
export const serviceCatalogNode: PortalNode = new PortalNode({
    "portal_id": servicePortal, "path_name": "catalog", "page_id": catalogPage, "parent_node_id": null,
    "page_id_for_item": recordPage, "item_table_id": itsmService, "item_parent_category_column_id": categoryColumn,
    "category_table_id": itsmServiceCategory, "category_parent_column_id": parentCategoryColumn, "active": true, "policy": "Open",
});
```

Практики: планировать иерархию заранее; `sys_order` шаг 100; `path_name` — лаконичный snake/kebab-case; `active: false` вместо удаления.
