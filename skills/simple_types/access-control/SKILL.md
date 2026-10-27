---
description: Стандарты управления доступом — роли, ACL, матрица доступов. Загружать при работе с access-control/roles.ts или security-acls/** (плюс script-rules для ACL-скриптов).
paths: '**/access-control/**/*'
user-invocable: false
---

# Управление доступом (Roles, ACL)

> ⚠️ Создание/редактирование ACL и ролей требует роли `security_admin` на стенде.

DSL-экспорт: TypeScript для метаданных, JavaScript для ACL-скриптов. Классы: `SysRole`, `SysUserCriteria`, `SysSecurityAcl`. `sys_id` — только в экспорте, при создании убрать.

Docs: [Roles](https://docs.simpleone.ru/platform/admin-help/access-control/user-roles/) · [ACL](https://docs.simpleone.ru/platform/admin-help/access-control/acl/). См. также `user-criteria`, `script-rules` §5.

## Логика ACL (⚠️ КРИТИЧНО)

- **Нет ACL = доступ разрешён (⚠️)**: если для операции (`read`/`write`/`create`/`delete`) над таблицей/полем нет ни одного ACL — действие разрешено всем. Запрет создаётся только наличием ACL (роль/условие/`answer = false`), не «отсутствием правила».
- **Несколько ACL** на одну операцию (`read`/`write`/`create`/`delete`) над таблицей/полем → **ИЛИ**: достаточно, чтобы хотя бы один разрешил.
- **Условия внутри одного ACL** → **И**: `role_ids` совпадает И `condition` выполняется И `script` вернул true.

**Модифицировать vs создавать новый ACL:**
- Та же логика, ещё одна роль → **модифицировать** (добавить роль в `role_ids`).
- Другая логика проверки → **новый** ACL с той же `operation + table_id + column_id`.
- «Роль ИЛИ владелец» → **два** ACL (платформа объединит по ИЛИ). «Роль И владелец» → **один** ACL со скриптом `answer = ss.hasRole('admin') && current.getValue('owner') === ss.getUserID()`. Не объединять разную логику через `||` в одном скрипте.

⚠️ Проверять совокупный эффект: при ИЛИ-логике (несколько ACL) любой, прошедший хотя бы один, получит доступ.

**`role_ids`**: `[]` = все авторизованные пользователи (НЕ «запретить всем» — для запрета `answer = false` в скрипте); `[role1, role2]` = только указанные роли.

## Запрет изменения поля (⚠️ КРИТИЧНО)

Требование «поле нельзя изменять» (вообще / после создания / всем кроме роли) закрывается **Field write-ACL**, а не чекбоксом `read_only` на колонке: чекбокс блокирует поле для всех, включая скрипты, которые его заполняют. Канон выбора механизма — `data-structure` («Нередактируемое поле: чем закрывать»).

⚠️ Помнить про ИЛИ-логику: если на ту же связку `write` + `table_id` + `column_id` уже есть разрешающий ACL — запрет не сработает. Перед добавлением проверить существующие ACL этой колонки (включая ACL с `any_fields`).

```typescript
// security-acls/pda_project/records.ts — поле нельзя менять никому, включая admin
import aclScript from "./write-pda-project-pda-number.js";   // answer = false;
export const writePdaProjectPdaNumber: SysSecurityAcl = new SysSecurityAcl({
    "operation": "write", "table_id": pdaProject, "column_id": pdaNumber, "role_ids": [],
    "admin_overrides": false, "active": true, "condition": null,
    "script": aclScript as unknown as string,
    "name": "[Write].pda_project.pda_number", "policy": "Open",
});
```

Закрытие по статусу — тот же ACL с `condition` (`"(state=closed)"`) и скриптом `answer = ss.hasRole('admin')`. Для формы дополнительно нужен клиентский скрипт `s_form.setReadOnly(...)`: ACL защищает данные на сервере, скрипт даёт пользователю понятное поведение поля.

## ACL-скрипты

Серверный контекст, ограниченные глобалы: `current` (`SimpleRecord`), `ss` (`SimpleSystem`), `answer` (boolean — **единственный способ вернуть результат**). **Запрещено**: `return` (ошибка), любые комментарии — `//`, `/* */`, JSDoc (`project-core-rule` → «Комментарии»), `SimpleRecord.query()` в циклах.

Проверка владельца записи и проверка роли:

```javascript
answer = current.getValue('owner') === ss.getUserID();
```

```javascript
answer = ss.hasRole('admin');
```

## Размещение и поля

- **Roles** — `SysRole`, единый `access-control/roles.ts`. Поля: `name`✱ (snake_case, уникально), `description`, `elevated_privilege`, `policy`.
- **ACL** — `SysSecurityAcl`, `access-control/security-acls/{table.name}/records.ts` (+ `.js` скрипты рядом, имя `toKebabCase(name, 50)`). Поля: `operation`✱, `table_id`✱, `column_id` (Field ACL, иначе null), `role_ids`✱, `any_tables`/`any_fields`, `admin_overrides`, `active`, `condition` (Encoded Query, **не** JS; пусто → `null`, не `""`; напр. `"active=true^state!=closed"`), `script` (импорт из `.js` или null), `name` (`[Operation].table` или `[Operation].table.column`), `policy`. Константа — camelCase от операции+таблицы.

```typescript
// security-acls/pda_project/records.ts — ACL по роли (без скрипта)
import { SysSecurityAcl } from "../../../../../platform/data-structure/columns/sys_security_acl";
import { pdaUser } from "../../roles";
import { pdaProject } from "../../../data-structure/tables/pda_project";
export const readPdaProject: SysSecurityAcl = new SysSecurityAcl({
    "operation": "read", "role_ids": [pdaUser], "table_id": pdaProject,
    "admin_overrides": true, "active": true, "condition": null, "script": null,
    "name": "[Read].pda_project", "column_id": null, "policy": "Open",
});

// ACL со скриптом: "script": aclScript as unknown as string, "role_ids": []
// roles.ts
export const pdaAdmin: SysRole = new SysRole({
    "name": "pda_admin", "elevated_privilege": false, "description": "...", "policy": "Protected",
});
```
