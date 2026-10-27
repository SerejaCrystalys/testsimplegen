---
description: Анализ Figma-макетов и конвертация их в виджеты SimpleOne — извлечение элементов, токенов и логики взаимодействия. Загружать при генерации/правке виджетов и портальных страниц; при Figma-ссылке в постановке или чате подгружается по имени из spec-author / spec-reviewer / code-author.
paths:
  - '**/widgets/**/*'
  - '**/portal/**/*'
user-invocable: false
---

# Figma-макеты и виджеты SimpleOne

Три режима: **1.** анализ макета для спецификации (`spec_create`/`spec_check`/чат — §A); **2.** генерация нового виджета (§0–11); **3.** модификация существующего виджета (§4, самый частый сценарий — перед правками **обязательно** изучить все файлы виджета: `record.ts`, `template.html`, `client_script.js`, `server_script.js`, `style.css`).

## Критические правила (DO / DON'T)

| ✅ DO | ❌ DON'T |
|---|---|
| Извлекать реальные значения из Figma (MCP или визуально) | Угадывать цвета/размеры; писать «примерные» стили |
| CSS-переменные `var(--token)` | Хардкод `#3B82F6`, `rgb(...)` |
| БЭМ `.block__element--modifier` с префиксом виджета | Произвольные/глобальные классы `.header`, `.my-btn-blue` |
| SimpleTags (`<button>`, `<reference>`, `<select>`…) | Нативные `<input>`/`<select>` |
| SVG с `currentColor` | SVG с захардкоженным `fill="#333"` |
| `simple-show` / `simple-if` для видимости | `style="display:none"` |
| Fluid: `width:100%`, `max-width` | Фиксированная `width:600px` |
| `transition` на конкретных свойствах | `transition: all` |
| Контент через `{data.field}` / `{options.key}` | Хардкод текста в шаблоне |
| Изучить код виджета и следовать его action-модели перед модификацией | Переписывать виджет «с нуля» |

## Figma MCP

Имена инструментов зависят от IDE и способа подключения: `mcp_figma_get_design_context`, `mcp__figma__get_design_context`, `mcp__plugin_figma_figma__get_design_context` — один и тот же инструмент. Ниже — короткие имена; брать тот, что доступен.

**Порядок вызова** (все инструменты требуют `fileKey` + `nodeId`):
1. **Параллельно (обязательно)**: `mcp_figma_get_design_context` (UI-код, компоненты, URL ассетов, иерархия) + `mcp_figma_get_variable_defs` (токены: цвета, шрифты, отступы).
2. Если `get_design_context` обрезан: `mcp_figma_get_metadata` (XML-дерево узлов) → найти sub-node ID → повторить `get_design_context` для него.
3. Опционально: `mcp_figma_get_screenshot` — только визуальное подтверждение, **не** единственный источник анализа.

❌ Несуществующие инструменты (не использовать): `get_node_info`, `get_styles`, `read_my_design`, `scan_nodes_by_types`, `export_node_as_image`, `get_local_components`.

**Если MCP недоступен**: запросить скриншот, визуально определить цвета/типографику/отступы/компоненты/иконки, уточнить у пользователя hex-цвета, SVG, шрифты, состояния.

**Режим ultra** (`ultra-rules`): после «Поехали» пользователя не спрашивать. Способ берётся из `ultra.options.figma`, который передаёт оркестратор: `mcp` — как выше; `screenshots` — скриншоты из `_local/figma/` и ответы интервью; `skip` — стили продукта. Чего не хватает — значение по стилям продукта и `ULTRA_ASSUMPTIONS` с пометкой `[макет]`. MCP перестал отвечать посреди прогона → так же, по скриншотам или стилям, с допущением `[макет]`.

**Парсинг ссылки**: `figma.com/design/<FILE_KEY>/...?node-id=<NODE_ID>`. node-id формата `X:Y` нормализовать к `X-Y`.

**Маппинг ответов → SimpleOne**: цвета → CSS vars (§1); типографика → шрифты (§2); отступы → spacing (§3); компоненты → SimpleTags (§5); React/Tailwind → plain HTML+CSS (§10); URL ассетов → SVG (§8).

---

## A. Анализ макета для спецификации

Сканировать вход на Figma-ссылки (`[(см. макет)](...figma.com...node-id=...)`). При обнаружении — извлечь FILE_KEY/NODE_ID, получить данные (MCP или скриншот), задокументировать в `_spec.md`.

**Извлекаемые элементы**: структура страницы (секции, блоки); отображаемые данные (поля/колонки записей → `column_name`); навигация (меню, вкладки, ссылки); интерактивные элементы (кнопки, UI-действия); встроенные виджеты/компоненты; текстовые элементы (заголовки, плейсхолдеры, сообщения); группировка (карточки, списки, таблицы, сетки).

**Логика взаимодействия** (из нескольких состояний макета): навигационные переходы; визуальная обратная связь; условная видимость; переходы между состояниями; состояния элементов (default/hover/active/disabled/selected); сообщения пользователю.

**Формат документирования:**
```markdown
На [странице/форме] [таблицы] (`table_name`) система транслирует [(см. макет)](ссылка):
* заголовок колонки «[Название]» (`column_name`);
* [интерактивный элемент] [(см. тексты)](texts.md#якорь).

При [действии] на [форме] [таблицы] (`table_name`) система:
* [визуальное изменение] [(см. макет)](ссылка);
* транслирует [новый элемент] [(см. перевод)](texts.md#якорь).
```

**Чек-лист**: ссылки обнаружены · элементы извлечены · логика взаимодействия описана · элементы сопоставлены с `column_name` · интерактивные → UI-действия/виджеты · текст → раздел локализации · результаты в `_spec.md`.

---

## 1. Дизайн-токены

Не хардкодить. Извлечь из макета → определить семантику → сопоставить с CSS-переменной платформы. Нет аналога → значение из макета с `/* TODO: добавить как --brand-accent */`.

| Семантика | CSS-переменная |
|---|---|
| Основной/вторичный фон | `--primary-background` / `--secondary-background` |
| Поверхности (главная/вложенная/подложка) | `--surface-primary` / `--surface-secondary` / `--surface-tertiary` |
| Текст (заголовки/подписи/плейсхолдер/ссылка/disabled) | `--text-primary` / `--text-secondary` / `--text-tertiary` / `--text-link` / `--text-disabled` |
| Бордеры (станд./hover/focus/error) | `--border-color` / `--border-color-hover` / `--border-color-focus` / `--border-color-error` |
| Акцент (станд./hover/active) | `--accent-color` / `--accent-color-hover` / `--accent-color-active` |
| Статусы | `--success-color` / `--warning-color` / `--error-color` / `--info-color` |
| Тени (малая/средняя/большая) | `--shadow-sm` / `--shadow-md` / `--shadow-lg` |

Имена токенов Figma маппятся по семантике: `Surface/Primary`→`--surface-primary`, `Text/Primary`→`--text-primary`, `Border/Default`→`--border-color`, `Accent/Default`→`--accent-color`, `Status/Success`→`--success-color`.

## 2. Типографика

Шрифты: **Montserrat** — заголовки H1–H6; **Open Sans** — body, лейблы, кнопки. Если в макете другие шрифты — использовать их. Замена шрифтов из MCP: Inter/Roboto/Helvetica/Arial → Open Sans; SF Pro/SF Display/System → Montserrat; моноширинный → `monospace`.

| Стиль | Размер/вес/line-height |
|---|---|
| H1 / H2 / H3 | 24px·700·1.3 / 20px·600·1.3 / 16px·600·1.4 |
| Body / Body Bold | 14px·400·1.5 / 14px·600·1.5 |
| Small / Caption / Button | 12px·400·1.5 / 11px·400·1.4 / 14px·600·1.4 |

## 3. Отступы и скругления

Базовая единица **4px**: xs `4` · sm `8` · md `12` · lg `16` · xl `20` · 2xl `24` · 3xl `32`. Скругления: badges/chips `2px`, кнопки/инпуты `4px`, карточки/панели `8px`, модалки `12px`, круглые `50%`.

---

## 4. DSL-структура виджетов

Файлы в `<product>/domain/business-logic/widgets/<widget-name>/` (имена **фиксированы**): `record.ts` (метаданные `SysWidget`), `template.html`, `style.css`, `client_script.js`, `server_script.js`, `instances.ts` (опц., экземпляры).

### 4.1. record.ts

```typescript
/// <reference path="../../../asset-modules.d.ts" />
import { someTable } from "../../../data-structure/tables/some_table";
import { SysWidget } from "../../../../../platform/data-structure/columns/sys_widget";
import myWidgetClientScript from "./client_script.js";
import myWidgetServerScript from "./server_script.js";
import myWidgetTemplate from "./template.html";
import myWidgetCss from "./style.css";

export const myWidget: SysWidget = new SysWidget({
    "css": myWidgetCss as unknown as string,
    "policy": "Protected",
    "name": "My Widget",
    "table_ids": [someTable],            // массив таблиц или null
    "active": true,
    "demo_data": null,
    "client_script": myWidgetClientScript as unknown as string,
    "template": myWidgetTemplate as unknown as string,
    "description": "My Widget description",
    "inheritance": false,
    "server_script": myWidgetServerScript as unknown as string,
    // "sys_id": "..."  — ТОЛЬКО для существующих виджетов (из экспорта)! Новые — не указывать.
});
```

> **⚠️ Деплой**: билдер определяет изменения по diff `record.ts`. Если менялись только скрипты/шаблон/CSS, а `record.ts` нет — внести тривиальное изменение в `record.ts` (например, `description`), иначе билдер не обнаружит виджет.

### 4.2. Обёртки скриптов

Скрипты оборачиваются в `export default function server()` / `client()`. Код внутри обёртки **без** дополнительного отступа.

```javascript
export default function server() {
// верхний уровень: /*global SharedClass:readonly */, ss.importIncludeScript('Name'), константы
(() => {                                  // IIFE — роутер action'ов
    if (input.action === 'INIT')  { data.translations = getTranslations(); /* справочники, начальные UI-состояния */ }
    if (input.action === 'UPDATE'){ data.translations = input.translations; processAction(); }
})();
function processAction() { /* бизнес-логика */ }
function getTranslations() { const sm = new SimpleMessage(); return { /* переводы */ }; }
}

export default function client() {
(async () => {
    s_widget.setFieldValue('action', 'INIT');
    await s_widget.serverUpdate();
    s_widget_custom.send = async () => {  // обработчики событий — через s_widget_custom
        s_widget.setFieldValue('action', 'UPDATE');
        await s_widget.serverUpdate();
    };
    function formUILogic() { /* UI-логика (disabled, visibility) — ВНУТРИ IIFE */ }
})();
function showLoader(isShow = true) { /* DOM-хелперы — ВНЕ IIFE, но ВНУТРИ export default */ }
}
```

> Если виджет использует общую серверную логику — подключать через `ss.importIncludeScript('Name')` + `/*global Name:readonly */` для линтера.

### 4.3. Action-модель (клиент ↔ сервер)

```
Клиент: s_widget.setFieldValue(key, val)  →  Сервер: input.key
        s_widget.setFieldValue('action','X'); await s_widget.serverUpdate()  →  if (input.action==='X'){...}; data.field = value
        s_widget.getFieldValue('field')   ←  data.field
```

| Объект | Сторона | Назначение |
|---|---|---|
| `input` | сервер (чтение) | данные от клиента (`s_widget.setFieldValue`) |
| `data` | сервер (запись) → клиент/шаблон (чтение) | данные сервера для шаблона/клиента |
| `options` | шаблон `{options.key}` / сервер `options.key` | параметры экземпляра (Widget Instance Options, `instances.ts`) |
| `s_widget` | клиент | `setFieldValue`, `getFieldValue`, `serverUpdate`, `addTemplate` |
| `s_widget_custom` | клиент | обработчики событий, вызываемые из шаблона |

Типичные action'ы: `INIT` (переводы, справочники, начальные данные), `UPDATE` (сохранить/обновить записи), `TOGGLE` (показать/скрыть, проверки). **Переиспользование переводов**: при `UPDATE` не грузить заново — клиент передаёт обратно (`data.translations = input.translations`), избегая повторного `SimpleMessage`.

### 4.4. Widget API

> Полный маппинг классов по контекстам — `script-rules`; сигнатуры — `api-index/client.md` (клиент) и `api-index/server.md` (сервер).

Виджетные (client): `s_widget.setFieldValue(key,val)` (на сервер), `s_widget.getFieldValue(key)` (из `data`), `await s_widget.serverUpdate()` (вызов сервера), `s_widget.addTemplate(id, html, css, mode)` (вставка HTML), `s_widget_custom.handler` (обработчик template-событий).

Контекстные (по размещению): `s_form.getUniqueValue()`/`getTableName()` (форма), `s_list.getCheckedRow(table)`/`getTablesName()` (список), `s_modal.setShow(bool)` (модал), `s_go.reloadWindow()`/`open(url)` (любой).

> `SimpleRecordSecure` — операции от имени пользователя (массовые действия, обновление; `.canUpdate()`); `SimpleRecord` — внутренние запросы (choices, метаданные).

### 4.5. Контекст виджета (форма / список)

`s_form` доступен на форме, на списке выбросит ошибку — определять через try/catch:
```javascript
let isForm;
try { isForm = !!s_form.getUniqueValue(); } catch (e) { isForm = false; }
if (isForm) {
    s_widget.setFieldValue('isRelatedList', true);
    // switch (s_form.getTableName()) — маппинг parent→child таблиц (продукто-специфично)
} else {
    s_widget.setFieldValue('tableName', s_list.getTablesName()[0]);
    s_widget.setFieldValue('isRelatedList', false);
}
```

### 4.6. Локализация (SimpleMessage)

НЕ хардкодить текст в шаблоне. Все строки: `SimpleMessage` на сервере → `data.translations` → `{data.translations.key}` в HTML.

```javascript
function getTranslations() {
    const sm = new SimpleMessage();
    return {
        save: sm.getMessage('(ITSM) Complete'),
        title: sm.getMessage('(ITSM) Records selected: {number}', 'app', { number: `${input.selectedRows.length}` }),
    };
}
```
```html
<button buttonType="primary" event-click="s_widget_custom.send()">{data.translations.save}</button>
```
> Ключи и префикс `(ITSM)` — продукто-специфичны; в другом продукте свой префикс или без него.

### 4.7. Валидация (.trim()) — клиент + сервер

**Клиент** — состояние кнопки; **сервер** — защита от пробельных значений. Всегда `input.field && input.field.trim()` (проверка на null/undefined И на пустую строку из пробелов).

```javascript
// Клиент: formUILogic()
const workNote = s_widget.getFieldValue('workNote') || '';
s_widget.setFieldValue('isDisabled', !workNote.trim());

// Сервер: запись только при непустом значении (необязательные поля)
if (input.technicalClosureNotes && input.technicalClosureNotes.trim()) {
    recordSecure.technical_closure_notes = input.technicalClosureNotes;
}
```
```html
<button disabled="{data.isDisabled}" event-click="s_widget_custom.send()">{data.translations.save}</button>
```

### 4.8. Результирующее сообщение (success/warning/error)

```html
<div simple-if="{data.isInfoShown}">
    <sectionmessage adaptivity="desktop" title="{data.translations.statusMessage}"
        messageType="{data.messageType}" isClosureIconShown="false" id="my-widget__sectionmessage"></sectionmessage>
    <button buttonType="primary" event-click="s_go.reloadWindow()">{data.translations.close}</button>
</div>
```
```javascript
data.messageType = totalNotUpdated ? (totalSuccessful === 0 ? 'error' : 'warning') : 'success';
```

### 4.9. Модификация: куда вносить изменения

Изучить файлы → определить action-модель, data flow, UI-состояния (`simple-if`/`simple-show`) → вносить по существующим паттернам:

| Задача | Файл | Действие |
|---|---|---|
| Новое поле | `template.html` | SimpleTag с `model`, `label`, `simple-if` в нужной позиции |
| Перевод | `server_script.js` | ключ в `getTranslations()` через `sm.getMessage()` |
| Валидация | `client_script.js` + `server_script.js` | клиент `.trim()`→disabled; сервер `.trim()` перед записью |
| Серверная логика | `server_script.js` | в нужный `action`-блок |
| UI-состояние | `server_script.js` + `template.html` | `data.isXShown` на сервере + `simple-if` в шаблоне |
| Расширить таблицы | `record.ts` | импорт + в массив `table_ids` |
| Обработчик | `client_script.js` | `s_widget_custom.handler = async () => {...}` |

> После модификации скриптов убедиться, что `record.ts` тоже изменён (хотя бы тривиально).

---

## 5. SimpleTags

> Каталог: [Common](https://docs.simpleone.ru/platform/developer-help/developer-api/widget-api/simpletags/common-simpletags/) · [Portal](https://docs.simpleone.ru/platform/developer-help/developer-api/widget-api/simpletags/portal-simpletags/) · [Tag Attributes](https://docs.simpleone.ru/platform/developer-help/developer-api/widget-api/simpletags/tag-attributes). Полный каталог — `widgets` §14. Общие атрибуты для большинства тегов: `class`, `id`, `simple-if`, `simple-show`, `event-change`, `event-click` (ниже — только специфические). ★ — обязательный.

**Поля ввода:**

| Figma → SimpleTag | Ключевые атрибуты |
|---|---|
| Text Input → `<string>` | `model`★, `label`, `placeholder`, `isMandatory`, `readOnly` |
| Textarea → `<textarea>` | `model`★, `label`, `placeholder`, `isMandatory`, `readOnly`, `value` |
| Select → `<select>` | `model`★, `label`, `options`, `value`, `placeholder`, `isMandatory`, `radioButtonsMode` |
| Multi-select → `<multiselect>` | `model`★, `label`, `options`, `values`, `canExcludeAll` |
| Checkbox → `<checkbox>` | `model`★, `label`, `isMandatory`, `readOnly` |
| Toggle → `<toggle>` | `checked`, `text`, `boldText` |
| Button → `<button>` | `buttonType`, `disabled`, `hint`, `svg`/`img` |
| Reference → `<reference>` | `model`★, `tableName`★, `label`, `condition`, `isFixed`, `isMandatory`, `canCreate`, `canRead` |
| Multi-reference → `<list>` | `model`★, `tableName`★, `label`, `condition`, `isFixed`, `isMandatory`, `value` (Array) |
| Date / DateTime → `<date>` / `<datetime>` | `model`★, `label`, `beginDate`, `endDate` (диапазон), `placeholder`, `isMandatory`, `readOnly`, `value` |
| Days of Week → `<daysofweek>` | `model`★, `label`, `isMandatory`, `readOnly`, `value` |
| Duration → `<duration>` | `model`★, `label`, `isMandatory`, `readOnly`, `value` |
| Duration Input → `<durationinput>` | `value`, `maxValue`, `exclude` (Array), `alignRight`, `onChange` (⚠️ не `event-change`) |
| Condition Builder → `<conditions>` | `model`★, `tableName`★, `tableId`, `label`, `isMandatory`, `readOnly`, `value` |
| Code Editor → `<codemirror>` | `model`★, `label`, `options`, `event-change` |
| WYSIWYG / HTML → `<wysiwygeditor>` / `<htmlEditor>` | `model`★, `label`, `isMandatory`, `readOnly`, `value` |

> ⚠️ `<reference>`/`<list>`/`<conditions>`: таблица задаётся `tableName` (**не** `data-table`). `<durationinput>`/`<search>`: `onChange` вместо `event-change`.

**Контейнеры и составные:**

| Figma → SimpleTag | Ключевые атрибуты |
|---|---|
| Modal → `<modal>` | `model`, `isShow`, `title`, `doClose`, `isMandatory`, `isFullScreenMode` |
| Section Message → `<sectionmessage>` | `title`, `messageType` (`success`/`warning`/`error`/`info`), `adaptivity`, `isClosureIconShown`, `id` |
| Tooltip (клик) / Hint (наведение) → `<tooltip>` / `<hint>` | `className`, `content` / `text` |
| Form → `<Form>` | `tableName`★, `sysId`, `view`/`name`, `save`, `uiActions`, `userScripts`/`isUserScripts`, `titleHide` |
| Attachment → `<attachment>` | `tableName`, `recordId`, `model`, `isServicePortal` |
| Activity Feed → `<activityFeed>` | `tableName`, `sysId`, `config` (JSON), `isAlwaysOpened` (рядом с `<Form>` — `sysId`/`tableName` авто) |
| Text Label → `<LabelTag>` | `text`, `color`, `icon` (HTML), `tagClass`, `iconClass`, `textClass` |
| Record Popup → `<recordPopup>` | `viewName`★, `tableName`★, `recordId`★, `title`, `config` (JSON), `isOpenButtonVisible`, `isHeaderVisible` |
| Report → `<report>` | `reportId`★, `condition`, `class`, `style` |
| Search → `<search>` | `searchQuery`, `searchUrl`, `tsGroupId`, `onChange`, `size`, `placeholder`, `showTitle`, `itemPage`, `itemView` |
| Record List → `<recordList>` | `name`★, `tableName`★, `label`, `condition`, `perPage`, `listView`, `compactView`, `hideHeader`, `hideCondition`, `hideCounter`, … |

**Портальные:**

| Figma → SimpleTag | Ключевые атрибуты |
|---|---|
| Portal List → `<listitems>` | `tableName`★, `condition`, `listView`, `itemPage`, `itemView`, `perPage`, `displayColumnNumber`, `fixedCondition` |
| REM Widget / Form → `<rem>` / `<remform>` | `tableName`★, `modelId`, `recordId`, `isPortal`, `isUserScripts`, `style`; `<remform>` доп.: `name`, `saveButtonCaption`, `hideSaveButton` (см. `record-extension-model`) |
| Breadcrumbs → `<breadcrumbs />` | self-closing |
| Sidebar → `<SideMenu>` / `<sidemenu>` | `parentsDepth`★, `childrenDepth`★, `title`, `isFixed`, `offsetTop`, `includeCategories`, `includeItems` (PascalCase в DSL, lowercase в docs — оба рабочие) |
| Search Results → `<results>` | `searchQuery`, `tsGroupId`, `isTreeShown`, `quantity`, `itemView` |
| Preview List → `<previewList>` | `tableName`★, `condition`, `subjectColumn`, `dateColumn`, `stateColumn`, `elementCount`, `itemView` |
| Category (tree/flat) → `<category>` / `<categoryFlat>` | `tableName`, `nodeId`, `categorySubject`, `categoryDescription`, `categoryIcon`, `itemSubject`, `itemIcon` |
| Card → `<card>` | `name`, `url`, `shortDescription`, `svg`/`img` |
| Dropdown Menu → `<dropdownMenu>` | `isShow`, `size` (`low`/`large`), `doClose`, `model`, `events` |
| Authorization → `<authfull>` / `<authheader>` | `redirectTo`, `isRegistration`, `isResetPassword`, `isSwitchLang`, `logo` / `profileUrl` |

Прочие элементы (вне SimpleTags) — обычный HTML с БЭМ: Card→`<div class="w__card">`, Table→`<table class="w__table">`, List→`<ul class="w__list">`, Tabs/Progress аналогично.

`buttonType`: `primary` (основная) · `secondary` (обводка, Back/Cancel) · `approve` (зелёная) · `destructive` (красная).

### 5.1. Синтаксис атрибутов (⚠️ КРИТИЧНО — фигурные скобки)

| Атрибут | Синтаксис | Пример |
|---|---|---|
| `model` (двусторонняя привязка) | **БЕЗ** скобок | `model="data.workNote"` |
| `options`, `checked`, `disabled`, `label` | **С** `{data.xxx}` | `options="{data.choices}"` |
| `simple-if`, `simple-show` | **С** `{data.xxx}` | `simple-if="{data.isShown}"` |
| `event-*` | **БЕЗ** скобок (JS-выражение) | `event-click="s_widget_custom.send();"` |
| `isMandatory`, `boldText` | строка `"true"`/`"false"` | `isMandatory="true"` |
| `simple-class` | строка или условный объект | `simple-class="{'x--active': data.isActive}"` |
| `simple-style` | JS-объект (camelCase) | `simple-style="{display:'flex', gap:'8px'}"` |
| Текстовый контент | **С** `{data.xxx}` | `{data.translations.title}` |

> Платформа принимает и без кавычек (`simple-if={data.x}`); в кодовой базе встречаются оба. Новый виджет — с кавычками; модификация — стиль оригинала.

```html
<!-- ✅ -->  <textarea model="data.workNote" label="{data.translations.label}"></textarea>
<!-- ❌ -->  <textarea model="{data.workNote}"></textarea>   <select options="data.choices"></select>
```

### 5.2. События

`event-click` (клик), `event-change` (изменение), `onChange` (только `<durationinput>`/`<search>`, вместо `event-change`). Вызов: `s_widget_custom.handler()` или `window.s_widget_custom.handler()` (оба рабочие). Определение в client_script: `s_widget_custom.handler = async () => {...}` (рекомендуется) или через `window.s_widget_custom`. Модификация — стиль оригинала.

---

## 6. CSS

БЭМ `.block__element--modifier`, block = имя виджета в kebab-case. **Уникальный префикс обязателен** (CSS не scoped). Правила: только CSS-переменные для цветов (не hex/rgb); отступы кратны 4px; `transition` на конкретных свойствах (не `all`); plain CSS (не SCSS); без `!important` и `@import`; весь CSS — в одном `style.css`.

> При **модификации** виджетов с хардкодом цветов (`#0B0C0E`, `#fff` и т.п.) — сохранять стиль оригинала, не рефакторить без задачи. **Новый** виджет — только переменные.

```css
.widget-name { padding: 16px; background: var(--surface-primary); }
.widget-name__title { font-family: 'Montserrat', sans-serif; font-size: 16px; font-weight: 600; color: var(--text-primary); }
.widget-name__action { cursor: pointer; transition: background-color 0.2s ease; }
.widget-name__action:hover { background-color: var(--surface-tertiary); }
.widget-name__card--active { border-color: var(--accent-color); }
```

## 7. Управление видимостью

| Директива | Поведение |
|---|---|
| `simple-show="{data.x}"` | всегда в DOM, управляется CSS `display` |
| `simple-if="{data.x}"` | добавляется/удаляется из DOM |
| `simple-class` | динамический класс: `"my-class"` или `"{'x--active': data.isActive}"` |
| `simple-style` | инлайн-стили JS-объектом (camelCase): `"{display:'flex', gap:'8px'}"` |

Значение — data binding `{data.x}` (основное) или JS-выражение (`simple-show="new Date().getDay()===5"`). Синтаксис скобок — §5.1.

## 8. Иконки

URL из `downloadUrls` ответа `get_design_context` (нет — через `get_metadata`→node). SVG: `fill`/`stroke` → `currentColor`; убрать лишние атрибуты; корректный `viewBox`; размеры 16/20/24px; встраивать инлайн.

```html
<svg width="16" height="16" viewBox="0 0 16 16" fill="none"><path d="M..." stroke="currentColor" stroke-width="1.5"/></svg>
```

## 9. Эффекты

Минимально, конкретные свойства: кнопки `background-color 0.2s ease, border-color 0.2s ease`; карточки `box-shadow 0.2s ease, border-color 0.2s ease`; инпуты focus `border-color 0.2s ease, box-shadow 0.2s ease`; раскрытие `max-height 0.3s ease-out`.

---

## 10. Конвертация React / Tailwind → SimpleOne

| React/JSX | SimpleOne |
|---|---|
| `className` | `class` |
| `{cond && <C/>}` | `<div simple-if="{data.cond}">` |
| `{items.map(...)}` | данные на сервере → `{data.items}` / `addTemplate()` |
| `onClick`/`onChange={h}` | `event-click`/`event-change="s_widget_custom.h();"` |
| `<input value={v}/>` | `<string model="data.v"/>` |
| `<select>` | `<select model="data.v" options="{data.options}"/>` |
| `useState` | `s_widget.setFieldValue`/`getFieldValue` |
| `useEffect(fn,[])` | логика в IIFE `client_script.js` |

| Tailwind | CSS |
|---|---|
| `bg-white` / `bg-gray-50/100` | `--surface-primary` / `--surface-secondary`/`tertiary` |
| `bg-blue-500` / `bg-red-500` / `bg-green-500` | `--accent-color` / `--error-color` / `--success-color` |
| `text-gray-900/500/400` | `--text-primary`/`secondary`/`tertiary` |
| `border-gray-200` | `--border-color` |
| `rounded`/`rounded-lg`/`rounded-full` | `4px`/`8px`/`50%` |
| `p-1..p-4`, `gap-1..gap-4` | `4px..16px` |
| `flex`/`flex-col`/`items-center`/`justify-between` | `display:flex`/`flex-direction:column`/`align-items:center`/`justify-content:space-between` |
| `text-sm/base/lg` | `12px`/`14px`/`16px` |
| `font-semibold`/`font-bold` | `600`/`700` |
| `shadow-sm`/`shadow-md` | `--shadow-sm`/`--shadow-md` |

> Полностью удалять Tailwind-классы и inline-стили из шаблона — всё переносить в `style.css`.

## 11. Адаптивность

Виджет — 100% ширины контейнера. flexbox/CSS Grid. Без фиксированных ширин — `min-width`/`max-width` вместо `width`.

---

## Связанные правила

`widgets` (DSL-формат, полный Widget API и каталог SimpleTags) · `script-rules` (стандарты кода, серверный API) · [Widget API](https://docs.simpleone.ru/platform/developer-help/developer-api/widget-api/) · [SimpleTags](https://docs.simpleone.ru/platform/developer-help/developer-api/widget-api/simpletags) · [API Directives](https://docs.simpleone.ru/platform/developer-help/developer-api/widget-api/api-directives).
