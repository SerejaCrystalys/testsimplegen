---
description: Стандарты критериев доступа (User Criteria). Загружать при работе с access-control/user-criterias.ts; при настройке доступа в каталоге услуг подгружается из service-catalog по имени.
paths: '**/access-control/user-criterias.ts'
user-invocable: false
---

# Критерии доступа (User Criteria)

> ⚠️ Создание/редактирование требует роли `security_admin` на стенде.

DSL-экспорт, TypeScript. Класс `SysUserCriteria`. `sys_id` — только в экспорте. [Docs](https://docs.simpleone.ru/platform/admin-help/access-control/user-criteria/).

## Назначение

Динамические группы пользователей по атрибутам (роли, группы, департаменты, компании) или скрипту — дополнительный уровень фильтрации доступа. В отличие от ролей (статическая принадлежность) описывает **условие соответствия** пользователя. Сценарии: видимость каталога услуг, статей БЗ, виджетов/дашбордов. Используется в ACL и через `SysRmcCategoryUserCriteria`/`SysRmcModelUserCriteria` (см. `service-catalog`).

## Размещение и поля

`SysUserCriteria`, единый `access-control/user-criterias.ts` (все критерии продукта). Константа — camelCase от `name`. `name` — глобально уникально (snake_case); проверять в `platform/`/`sa/`/`<product>/`.

Поля: `name`✱, `active`, `advanced` (true → скриптовая логика), `condition` (Encoded Query или **inline-скрипт** при `advanced: true` — отдельных `.js` нет), `match_all` (`true`=И / `false`=ИЛИ), `roles`/`users`/`groups`/`companies`/`departments` (ссылки), `policy`.

Порядок деплоя: **после** `sys_role`, **до** связующих сущностей каталога (`sys_rmc_category_user_criteria`, `sys_rmc_model_user_criteria`).

```typescript
import { SysUserCriteria } from "../../../platform/data-structure/columns/sys_user_criteria";
import { pdaAdmin } from "./roles";

export const projectManagers: SysUserCriteria = new SysUserCriteria({
    "name": "project_managers", "active": true, "advanced": false, "condition": null,
    "match_all": false, "policy": "Open", "roles": [pdaAdmin], "users": null,
});

// advanced: inline-скрипт в condition, результат в answer
export const activeProjectMembers: SysUserCriteria = new SysUserCriteria({
    "name": "active_project_members", "active": true, "advanced": true,
    "condition": "var gr = new SimpleRecord('pda_team_member'); gr.addQuery('user_id', ss.getUserID()); gr.addQuery('active', true); gr.query(); answer = gr.next();",
    "match_all": false, "policy": "Open", "roles": null, "users": null,
});
```

## Практики

`advanced: false` с атрибутными фильтрами — когда достаточно стандартных полей; `advanced: true` — только для сложной динамики. Один критерий = одно логическое условие (переиспользовать, не дублировать). Деактивация через `active: false` безопаснее удаления. `match_all: true` для строгих, `false` для разрешительных.
