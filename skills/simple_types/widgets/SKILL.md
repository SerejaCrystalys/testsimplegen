---
description: Стандарты виджетов и их инстансов (template, CSS, server, client). Загружать при работе с business-logic/widgets/**, генерации виджета или Figma-ссылке (вместе с design-system-rules).
paths: '**/business-logic/widgets/**/*'
user-invocable: false
---

# Виджеты (Widgets, Widget Instances)

DSL-экспорт: TypeScript (метаданные), JS (логика), HTML (шаблон), CSS (стили). Классы: `SysWidget`, `SysWidgetInstance`. `sys_id` — только в экспорте.

> 🎨 Дизайн, обёртки скриптов, action-модель (`input`/`data`/`s_widget`/`s_widget_custom`), Widget API, каталог SimpleTags, директивы, БЭМ/CSS — `design-system-rules`. Правила скриптов — `script-rules`. Здесь — DSL-структура и встраивание.

## Контекст выполнения (⚠️ КРИТИЧНО)

| Файл | Контекст | Глобалы | Запрещено |
|------|----------|---------|-----------|
| `server_script.js` | сервер | `input`, `data` + серверный API | `s_widget`, `s_form`, `current` |
| `client_script.js` | клиент | `s_widget`, `s_widgets` + клиентский API | `ss`, `current`, `input`, `data` |
| `template.html` | шаблон | `simple-show`/`simple-if`/`data-*` | `style="display:none"`, `innerHTML`, `document.*` |

Обмен между контекстами — только `s_widget.setFieldValue()` + `s_widget.serverUpdate()` (клиент→сервер) и `data` (сервер→клиент). Полный API по контексту — `script-rules`.

**Внешние библиотеки в `client_script.js`**: подключаются загрузкой с CDN. Не подключать втихую — предложить через `NEEDS_USER_INPUT`. Политика и серверный вариант (вендоринг Include) — [script-rules.mdc → Внешние библиотеки](../script-rules.mdc#внешние-библиотеки).

## Порядок импорта (⚠️ нарушение → «… must be Integer»)

`SysWidget` → `SysWidgetInstance` (`widget_id`) → `SysUiFormElement` (`widget_instance_id`).

Уникальность: `sys_widget.name` глобально; `sys_widget_instance` — `widget_id` + `page_column_id` (желательно).

## Размещение и поля

Папка `business-logic/widgets/{widget-name}/`: `record.ts` (`SysWidget`), `instances.ts` (`SysWidgetInstance`), `client_script.js`, `server_script.js`, `template.html`, `style.css`. Константа — `toCamelCase(name, 50)`.

**`SysWidget`**: `name`✱ (уникально), `description`, `client_script`/`server_script`/`template`/`css` (импорты с `as unknown as string`), `table_ids` (`null` — универсальный / `[tableRef]` — ограничен таблицами), `active`, `inheritance`, `demo_data` (JSON опций), `policy`. Требует `/// <reference path="../../../asset-modules.d.ts" />` для импорта `.html`/`.css`.

**`SysWidgetInstance`**: `widget_id`✱, `page_column_id` (`null` для виджетов на формах — встраивание через `SysUiFormElement`; ссылка на колонку для портальных), `sys_order`, `option_schema_values` (JSON конфигурации), `active`, `public`, `policy`.

**`description` = JSDoc опций**: заполнять `description` назначением виджета + перечнем опций (`option_schema_values` / параметры `input`) в стиле JSDoc, напр. `@option record_id {string} sys_id demo_story (обязателен)`, `@option theme {string} 'light'|'dark'; default 'light'`.

```typescript
// record.ts
/// <reference path="../../../asset-modules.d.ts" />
import { SysWidget } from "../../../../../platform/data-structure/columns/sys_widget";
import clientScript from "./client_script.js";
import serverScript from "./server_script.js";
import template from "./template.html";
import css from "./style.css";
export const demoStageModel: SysWidget = new SysWidget({
    "name": "Demo Stage Model", "description": "Stage progression for demo stories",
    "table_ids": null, "active": true, "inheritance": false, "demo_data": null, "policy": "Open",
    "css": css as unknown as string, "client_script": clientScript as unknown as string,
    "server_script": serverScript as unknown as string, "template": template as unknown as string,
});

// instances.ts
import { demoStageModel } from "./record";
export const demoStageModelInstance: SysWidgetInstance = new SysWidgetInstance({
    "widget_id": demoStageModel, "page_column_id": null, "sys_order": 1,
    "option_schema_values": null, "active": true, "public": true, "policy": "Open",
});
```

## Скрипты (пример input/data flow)

```javascript
// server_script.js — НЕ current/previous; читать input, заполнять data
export default function server() {
  (function() {
    if (!input.record_id) return;
    const record = new SimpleRecord('demo_story');
    record.get(input.record_id);
    if (input.action === 'UPDATE_STAGE' && input.new_stage) {
      record.demo_stage = input.new_stage;
      record.update();
    }
    data.current_stage = record.getValue('demo_stage');
  })();
}

// client_script.js — сначала отправить параметры + serverUpdate, затем читать data
export default function client() {
  (async () => {
    const recordId = s_form.getUniqueValue();
    if (!recordId) return;
    s_widget.setFieldValue('record_id', recordId);
    await s_widget.serverUpdate();
    const stage = s_widget.getFieldValue('current_stage');   // рендер по data
  })();
}
```

## Платформенные ограничения

Числовые ключи `data`-объектов (18-значные sys_id) → PHP-ошибка `Array index exceeds maximum supported bound` — массив объектов `[{id: String(sys_id), ...}]`, не map. DOM-атрибуты `data-*` устаревают после серверного обновления — обновлять из ответа.

## Встраивание на форму (⚠️ КРИТИЧНО)

`SysWidget` без `SysWidgetInstance` и `SysUiFormElement` **не отображается нигде**. Для формы создавать все три:
1. `SysWidget` → `record.ts`
2. `SysWidgetInstance` → `instances.ts` (`page_column_id: null`)
3. `SysUiFormElement` → `forms/<table>/elements.ts` (`widget_instance_id`, `column_id: null`)

```typescript
import { demoStageModelInstance } from "../../business-logic/widgets/demo_stage_model/instances";
export const stageModelWidget: SysUiFormElement = new SysUiFormElement({
    "form_section_id": stageModelSection, "widget_instance_id": demoStageModelInstance,
    "column_id": null, "position": 1,
});
```
Подробнее — `forms`.
