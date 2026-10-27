---
description: Стандарты системных свойств и их категорий. Загружать при работе с business-logic/system-properties.ts или business-logic/property-categories.ts.
paths:
  - '**/business-logic/system-properties.ts'
  - '**/business-logic/property-categories.ts'
user-invocable: false
---

# Системные свойства (System Properties)

DSL-экспорт, TypeScript. Класс `SysProperty`. `sys_id` — только в экспорте. [Docs](https://docs.simpleone.ru/platform/admin-help/system-settings/system-properties/).

Размещение: единый `business-logic/system-properties.ts`. Константа — camelCase от `name`. `name` — глобально уникально, dot-notation; проверять в `platform/`/`sa/`/`<product>/`.

**Поля**: `name`✱ (формат `<prefix>.<module>.<property_name>`, lowercase, напр. `sdlc.notification.logo_href`), `value` (**всегда строка** — boolean как `"true"`, число `"100"`, JSON экранированный, список `"a,b,c"`), `description`, `property_category_id` (или null), `public` (доступно на клиенте), `policy`. Порядок импорта: не зависит от других типов.

```typescript
import { SysProperty } from "../../../platform/data-structure/columns/sys_property";

export const sdlcNotificationLogoHref: SysProperty = new SysProperty({
    "property_category_id": "156076775602982123", "name": "sdlc.notification.logo_href",
    "public": false, "value": "",
    "description": "Specify the image URL to display as logo in the notification header", "policy": "Open",
});

// JSON-значение — экранированная строка
export const docsUrl: SysProperty = new SysProperty({
    "property_category_id": null, "name": "sdlc.documentation.task_types_url", "public": false,
    "value": "'{\"ru\":\"https://docs.simpleone.ru/ru/...\",\"en\":\"https://docs.simpleone.ru/...\"}'",
    "description": "", "policy": "Open",
});
```
