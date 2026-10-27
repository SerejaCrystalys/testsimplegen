---
description: Стандарты UI-действий — кнопки, ссылки, пункты контекстного меню (серверные и клиентские). Загружать при работе с business-logic/ui-actions/** (плюс script-rules).
paths: '**/ui-actions/**/*'
user-invocable: false
---

# UI-действия (кнопки, ссылки, контекстное меню)

DSL-экспорт: TypeScript + JavaScript-скрипты. Класс `SysUiAction`. `sys_id` — только в экспорте. [Docs](https://docs.simpleone.ru/platform/developer-help/business-logic/ui-actions/), правила скриптов — `script-rules` §8.

## Контекст выполнения

`client: true` → клиент (`s_form`, `s_user`); `client: false` → сервер (`current`, `ss`, + `__resolveServerResponse()` при `wait_server_response: true` — **обязательно** вызвать в конце, иначе UI заблокируется). Не смешивать контексты.

```javascript
// client: export default function script() { s_form.save(); }
// server: (function() { /* логика */ __resolveServerResponse(); })();
```

## Размещение и поля

`business-logic/ui-actions/{table.name}/records.ts` (+ `.js` рядом, имя `toKebabCase(name, 50)`). Константа — camelCase от `name`. Уникальность (желательно): `name`+`table_id`, `action_name`+`table_id`.

**Ключевые поля**: `name`✱, `table_id`✱ (⚠️ **НЕ null** — даже для «глобальных» действий списка указывать конкретную таблицу + флаг размещения; null делает action невалидным), `client`, `script` или `url`, `wait_server_response`, `order`, `condition`, `show_insert`/`show_update`, `active`, `form_style`/`list_style` (`Unstyled`/`Primary`/`Secondary`), `inherits`, `overrides_by` (при переопределении — родитель должен существовать), `policy`.

**`condition`** — **JavaScript-выражение** (НЕ Encoded Query!), возвращает `true`/`false`, клиентский контекст: `"current.sys_id ? current.canUpdate() : current.canCreate();"`. Пусто → `""`, не `null`.

**Скрытие наследуемой кнопки (⚠️)**: `active: false` на оверайде отключает сам оверайд (он не применится) — родительская кнопка остаётся видимой. Это НЕ деактивация кнопки. Чтобы скрыть унаследованную кнопку — переопределить её на уровень скрытия и в `condition` прописать `false`.

**Размещение** (булевы флаги): на форме — `form_button`, `form_context_menu`, `form_link`, `form_header_button`, `form_bottom_button`, `form_field_label_context_menu`. На списке:

| Флаг | Когда |
|------|-------|
| `list_burger_menu` | меню списка ☰: переключение представления (календарь/воронка/Kanban), массовые операции, экспорт |
| `list_kebab_menu` | меню строки ⋮: действия над **одной записью** (открыть/копировать/удалить) |
| `list_banner_button` | создание записи, основные действия |
| `list_context_menu` / `list_header_right_button` / `list_bottom_button` | настройки списка / быстрые действия / пагинация |
| `list`, `related_list` | отображать на списке / связанных списках |

⚠️ Переключение представления → `list_burger_menu`, **не** `list_kebab_menu`. Спец-типы для полей: `reference_field_type`, `list_field_type`, `document_id_field_type`, `m2m_type`, `scripted_list`, `simple_tag_record_list`.

```typescript
import { SysUiAction } from "../../../../platform/data-structure/columns/sys_ui_action";
import { pdaProject } from "../../data-structure/tables/pda_project";
import saveScript from "./save.js";
export const save: SysUiAction = new SysUiAction({
    "name": "Save", "table_id": pdaProject, "client": true, "active": true, "order": -99,
    "show_insert": true, "show_update": true, "form_button": true, "form_bottom_button": true,
    "wait_server_response": true, "form_style": "Unstyled", "list_style": "Unstyled",
    "condition": "current.sys_id ? current.canUpdate() : current.canCreate();",
    "url": "", "overrides_by": "155931135900000504", "policy": "Open",
    "script": saveScript as unknown as string,
    // прочие флаги размещения — false по умолчанию
});
```
