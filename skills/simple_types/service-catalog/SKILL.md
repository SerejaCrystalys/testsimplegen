---
description: Стандарты каталога услуг — каталоги, категории, модели запросов, критерии доступа. Загружать при работе с service-catalog/**.
paths: '**/service-catalog/**/*'
user-invocable: false
---

# Каталог услуг (RMC)

Request Model Catalog на базе REM. DSL-экспорт, TypeScript. Классы: `SysRmcCatalog`, `SysRmcCategory`, `SysRmcModel`, `SysRmcCategoryUserCriteria`, `SysRmcModelUserCriteria`. `sys_id` — только в экспорте. Каждая таблица — в отдельном файле `service-catalog/{tableName}.ts`. Константа — `toCamelCase(name||title, 50)`. [Docs](https://docs.simpleone.ru/platform/service-tools/service-portfolio-and-contracts/request-model-catalog/rem-service-catalog), `user-criteria`.

## Архитектура и порядок импорта (⚠️ КРИТИЧНО)

```
SysRmcCatalog → SysRmcCategory (catalog_id; parent_id — иерархия) → [SysRmcCategoryUserCriteria (category_id, user_criteria_id), SysRmcModel (category_id, table_id)] → SysRmcModelUserCriteria (model_id, user_criteria_id)
```

`sys_user_criteria` создаются заранее (в `access-control/`). Целевая таблица модели (`table_id`) — до модели. Уникальность (желательно): catalog `name`; category `name`+`catalog_id`; model `title`+`category_id`; criteria — `category_id`/`model_id` + `user_criteria_id`.

## Поля

| Тип | Ключевые поля |
|-----|---------------|
| `SysRmcCatalog` | `name`✱, `description`, `admins` (List<employee>, роль `catalog_manager`), `active`, `policy` |
| `SysRmcCategory` | `name`✱, `description`, `order`, `catalog_id`✱, `parent_id` (иерархия; null — верхний уровень), `image`, `active`, `policy` |
| `SysRmcModel` | `title`✱, `description`, `order`, `table_id`✱ (где создаётся запрос — `task` или наследник), `category_id`✱, `active`✱, `icon`, `policy` (наследует от `SysReModel`→`SysReContainer`) |
| `SysRmcCategoryUserCriteria` | `category_id`✱, `user_criteria_id`✱, `user_access_type` (`"allow_access"`/`"deny_access"`), `policy` |
| `SysRmcModelUserCriteria` | `model_id`✱, `user_criteria_id`✱, `user_access_type`, `policy` |

Доступ на портале контролируется через User Criteria (allow/deny). Если пользователь не соответствует ни одному критерию элемента — категория не отображается; пустые категории тоже скрыты. Управление каталогом — роль `catalog_admin`/`admin`. Иерархия категорий: родитель объявляется в файле до дочерней.

Translated Text (через `sys_translation`, не SourceMessage): `name`/`description` у catalog/category, `title`/`description` у model.

## Примеры

```typescript
// sys_rmc_catalog.ts
export const serviceCatalog: SysRmcCatalog = new SysRmcCatalog({ "name": "Service Catalog", "description": "IT Services", "active": true, "policy": "Open" });
// sys_rmc_category.ts — родитель до дочерней
export const itAssistance: SysRmcCategory = new SysRmcCategory({ "name": "IT Assistance", "order": 100, "catalog_id": serviceCatalog, "parent_id": null, "active": true, "policy": "Open" });
export const hardwareRequests: SysRmcCategory = new SysRmcCategory({ "name": "Hardware Requests", "order": 100, "catalog_id": serviceCatalog, "parent_id": itAssistance, "active": true, "policy": "Open" });
// sys_rmc_model.ts
import { itsmServiceRequest } from "../data-structure/tables/itsm_service_request";
export const newLaptopRequest: SysRmcModel = new SysRmcModel({
    "title": "New Laptop Request", "description": "Request a new laptop", "order": 100,
    "table_id": itsmServiceRequest, "category_id": hardwareRequests, "active": true, "icon": null, "policy": "Open",
});
// sys_rmc_category_user_criteria.ts / sys_rmc_model_user_criteria.ts
import { itDepartmentUsers } from "../access-control/user-criterias";
export const itAssistanceAccess: SysRmcCategoryUserCriteria = new SysRmcCategoryUserCriteria({ "category_id": itAssistance, "user_criteria_id": itDepartmentUsers, "policy": "Open" });
export const laptopAccess: SysRmcModelUserCriteria = new SysRmcModelUserCriteria({ "model_id": newLaptopRequest, "user_criteria_id": itDepartmentUsers, "policy": "Open" });
```

**Портал**: каталог интегрируется через PortalNode (item table → `sys_rmc_model`, category table → `sys_rmc_category`, item parent → `category_id`, category parent → `parent_id`). Виджет View Task Readonly Form показывает REM-атрибуты созданной задачи.

Практики: 2–3 уровня иерархии; переиспользовать существующие `SysUserCriteria`; `order` шаг 100; у целевой таблицы должны быть формы/логика; `active: false` вместо удаления.
