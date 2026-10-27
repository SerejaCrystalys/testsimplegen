---
description: Стандарты бизнес-логики — Business Rules и Client Scripts. Загружать при работе с business-logic/business-rules/** или client-scripts/** (плюс script-rules для самих скриптов).
paths: '**/business-logic/**/*'
user-invocable: false
---

# Бизнес-логика (Business Rules, Client Scripts, Widgets)

DSL-экспорт: TypeScript для метаданных, JavaScript для логики. Классы: `SysBusinessRule`, `SysScriptClient`, `SysWidget`. `sys_id` — только в экспорте. Правила скриптов — `script-rules`; виджеты — `widgets`.

## Минимальная достаточность (⚠️ КРИТИЧНО)

BR и Client Scripts создавать **только при явной необходимости** и обосновывать каждый. НЕ создавать, если: нет указания в постановке; задача решается стандартно (`set_value`, ACL, Client Script); функциональность уже есть в `platform/`/`sa/`. Создавать при: явном требовании («при сохранении…», «автоматически заполнить…», «валидация на сервере»); сложной серверной логике (кросс-таблицы, async, интеграции); безопасности (валидация, необходимая на сервере). Если обоснование неочевидно — уточнить.

| Задача | Альтернатива BR |
|--------|------------------|
| Установить значение | `set_value` (BR без скрипта) |
| Скрыть/показать поле | Client Script `onLoad`/`onChange` |
| Ограничить доступ | ACL |
| Сделать поле нередактируемым | write-ACL (сервер) + `s_form.setReadOnly()` (форма) — **не** чекбокс `read_only`, если поле заполняется скриптом или закрыто по роли/статусу (`data-structure`, `access-control`) |
| Валидация формы | Client Script `onSubmit` |

## Контекст выполнения

| Тип | Контекст | Спец-глобалы |
|-----|----------|--------------|
| Business Rule (before/after/async) | сервер | `current`, `previous` |
| Client Script (onLoad/onChange/onSubmit) | клиент | `s_form`, + `fieldName`/`oldValue`/`newValue` (onChange) |
| Client Script (onCellEdit) | клиент | `s_list` |

Не смешивать контексты. Полный API — `script-rules`. Docs: [Business Rules](https://docs.simpleone.ru/platform/developer-help/business-logic/business-rule), [Client Scripts](https://docs.simpleone.ru/platform/developer-help/business-logic/client-scripts/), [Execution Order](https://docs.simpleone.ru/platform/developer-help/business-logic/execution-order).

## Business Rules

`SysBusinessRule`, путь `business-logic/business-rules/{table.name}/records.ts` (+ `.js` рядом, имя `toKebabCase(name, 50)`). Константа — camelCase от `name`. Скрипт: `export default function script() { (function executeRule(current, previous) { ... })(current, previous); }` (IIFE, серверный контекст). Ограничения — `script-rules` §6.

Поля: `table_id`✱, `when` (`before`/`after`/`async`), `action_insert`/`action_update`/`action_delete`, `filter_conditions` (Encoded Query; пусто → `"()"`, **не** `null`/`""`), `role_conditions` (пусто → `null`), `advanced` (true для скрипта), `set_value` (JSON-массив), `abort_action` + `message`, `script`, `order`, `priority`, `inheritance`, `name`, `policy`.

### filter_conditions (Encoded Query)

Всё выражение в скобках. **Имена колонок — `column_name`, не title/label.** Без пробелов.

| Операция | Формат |
|----------|--------|
| равно / не равно | `column=value` / `column!=value` |
| пусто / не пусто | `columnISEMPTY` / `columnISNOTEMPTY` |
| изменилось на / изменилось | `columnCHANGESTOvalue` / `columnCHANGES` |
| И / ИЛИ | `^` / `^OR` |
| динамический фильтр | `columnDYNAMICsys_id` |

Примеры: `"()"` (всегда), `"(demo_statusCHANGESTOcancelled)"`, `"(demo_groupISNOTEMPTY^demo_assigneeISEMPTY^ORdemo_groupISEMPTY^demo_assigneeISNOTEMPTY)"`.

## Client Scripts

`SysScriptClient`, путь `business-logic/client-scripts/{table.name}/records.ts` (+ `.js` рядом). Скрипт: **без IIFE** (платформа оборачивает сама) — прямой код в `export default function script() { }`. Многие методы `s_form` асинхронны — `await`. Детали и управление видимостью/обязательностью — `script-rules` §7.

Поля: `table_id`✱, `type` (`onLoad`/`onChange`/`onSubmit`/`onCellEdit`), `column_id`, `condition` (Encoded Query или `null`, **не** `""`), `views`, `script`, `order`, `inherited`, `name`, `policy`.

`column_id` — **ссылка на импортированный `SysDbColumn`, не строка**. Обязателен для `onChange`/`onCellEdit`; для `onLoad`/`onSubmit` — `null`.

```typescript
export const onChangeState: SysScriptClient = new SysScriptClient({
    "type": "onChange", "column_id": stateColumn,   // ссылка на объект, НЕ "state"
    "table_id": pdaProject, "condition": null, /* ... */
});
```

## Примеры

```typescript
// business-rules/pda_release/records.ts
import { SysBusinessRule } from "../../../../platform/data-structure/columns/sys_business_rule";
import { pdaRelease } from "../../data-structure/tables/pda_release";
import setDisplayName from "./set-displayable-name.js";
export const setDisplayableName: SysBusinessRule = new SysBusinessRule({
    "when": "before", "order": 999, "active": true, "table_id": pdaRelease,
    "filter_conditions": "()", "role_conditions": null, "advanced": true,
    "action_insert": true, "action_update": true, "action_delete": false,
    "abort_action": false, "set_value": "[]", "name": "Set displayable name",
    "policy": "Open", "script": setDisplayName as unknown as string,
});
```
```javascript
// set-displayable-name.js
export default function script() {
  (function executeRule(current, previous = null) {  // previous != null только при update
    const product = current.getDisplayValue('pda_product_id');
    current.setValue('pda_displayable_name', `${product} ${current.getValue('pda_major')}`);
  })(current, previous);
}
```

## Виджеты

Папка `widgets/<name>/`: `record.ts` (`SysWidget`), `client_script.js`, `server_script.js`, `template.html`, `style.css`, `instances.ts` (`SysWidgetInstance`). Полная спецификация — `widgets`, `design-system-rules`.

Платформенные ограничения: числовые ключи `data`-объектов (18-значные sys_id) → PHP-ошибка `Array index exceeds maximum supported bound` — использовать массив объектов `[{id: String(sys_id), ...}]`, не map. DOM-атрибуты `data-*` устаревают после серверного обновления — обновлять из ответа сервера.
