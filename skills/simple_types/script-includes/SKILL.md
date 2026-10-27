---
description: Стандарты подключаемых серверных библиотек (Script Includes). Загружать при работе с business-logic/script-includes/** (плюс script-rules).
paths: '**/script-includes/**/*'
user-invocable: false
---

# Script Includes

Серверные переиспользуемые скрипты (для Business Rules, других Script Includes, серверных скриптов). DSL-экспорт: TypeScript + JS. Класс `SysScriptInclude`. `sys_id` — только в экспорте. [Docs](https://docs.simpleone.ru/platform/developer-help/business-logic/script-includes/), `script-rules`.

Контекст — **серверный**. При `client_callable: true` вызывается с клиента через `SimpleAjax.getXML()`, результат — через `setResult(value)`; класс наследуется от `AbstractAjaxProcessor`.

## Размещение и поля

`business-logic/script-includes/records.ts` (+ `.js` рядом, имя `toKebabCase(name)`). Константа — camelCase от `name`. Порядок импорта — не зависит от таблиц.

Поля: `name`✱ (**имя класса**, PascalCase, **глобально уникально** — проверять в `platform/`/`sa/`/`<product>/`; совпадает с именем класса в скрипте), `client_callable`, `script`, `description`, `active`, `policy`.

```typescript
import { SysScriptInclude } from "../../../../platform/data-structure/columns/sys_script_include";
import taskUtilsScript from "./task-utils.js";
export const taskUtils: SysScriptInclude = new SysScriptInclude({
    "name": "TaskUtils", "description": "Утилиты для задач", "client_callable": false,
    "active": true, "policy": "Open", "script": taskUtilsScript as unknown as string,
});
```
```javascript
// task-utils.js — серверный
export default function script() {
  const TaskUtils = Class.create();
  TaskUtils.prototype = {
    initialize: function() {},
    calculateSLA: function(taskId) { const t = new SimpleRecord('task'); t.get(taskId); /* ... */ },
    type: 'TaskUtils',
  };
}

// client-callable — наследование от AbstractAjaxProcessor
export default function script() {
  const AjaxHelper = Class.create();
  AjaxHelper.prototype = Object.extendsObject(AbstractAjaxProcessor, {
    getUserInfo: function() { const id = this.getParameter('sysparm_user_id'); /* ... */ return JSON.stringify({}); },
    type: 'AjaxHelper',
  });
}
```

Практики: один Script Include — одна ответственность; выносить общую логику вместо дублирования в BR; чистые функции где возможно.
