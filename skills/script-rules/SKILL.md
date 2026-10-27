---
description: Канон серверных и клиентских скриптов SimpleOne — порядок чтения API-справочника (индекс сигнатур api-index/ по контексту, справочник целиком не грузить), запрещённые паттерны, SimpleOne ≠ ServiceNow. Загружать при создании или правке любого .js (Business Rule, Client Script, ACL-скрипт, UI Action, Script Include, Scheduled Script, Notification Script, контроллер виджета).
paths: '**/*.js'
user-invocable: false
---
# Правила написания скриптов SimpleOne

## API-справочник (⚠️ КРИТИЧНО)

**Единственный источник правды**: `simpleone-api.d.ts` — все допустимые классы, методы, глобалы и типы. Метода нет в справочнике → он **не существует** на платформе, использовать запрещено.

### Как читать справочник (⚠️ обязательный порядок)

Справочник целиком **не загружать** — 2500 строк, из них две трети JSDoc. Работать через индекс сигнатур `api-index/`, он покрывает то же множество API:

| Что пишем | Что грузить |
| --------- | ----------- |
| Серверный скрипт (BR, Script Include, Scheduled Script, ACL, UI Action server, Notification, server-часть виджета) | `api-index/server.md` + `api-index/globals.md` |
| Клиентский скрипт (Client Script, UI Action client, client-часть виджета) | `api-index/client.md` + `api-index/globals.md` |
| Скрипт с обеими частями (виджет целиком) | оба + `api-index/globals.md` |

Индекс даёт имена и сигнатуры — этого достаточно, чтобы проверить существование метода и его параметры. Нужна семантика, смысл параметра или пример — грепнуть символ в полном справочнике, **не** читая его целиком:

```bash
grep -n -A 12 "getDisplayValue" simpleone-api.d.ts
```

Если задачу решает метод из справочника — использовать его. Vanilla JS допустим **только** для базовых операций без платформенного аналога: строки, массивы (`map`/`filter`/`reduce`/…), `Object.*`, `JSON`, `Math`, `RegExp`, `try/catch`, `Promise`/`async-await`, `typeof`/`instanceof`/`Array.isArray`.

> **SimpleOne ≠ ServiceNow.** Метода нет в `api-index/` — не додумывать по аналогии с ServiceNow.

### Доступный API по контексту

Ниже — маппинг **класс/глобал → назначение**. Сигнатуры — в `api-index/server.md` / `api-index/client.md`.

**Серверный контекст** (Business Rules, Script Includes, Scheduled Scripts): `SimpleRecord` / `SimpleRecordSecure` (записи БД; Secure — с ACL), `ss` (`SimpleSystem`: системные операции, логирование, свойства, события), `SimpleDateTime`, `SimpleDuration`, `SimpleTime`, `SimpleAttachment` / `SimpleAttachmentService`, `sws` (`SimpleWebService`) → `SimpleRestRequest` / `SimpleRestResponse` (исходящие HTTP), `SimpleSchedule`, `SimpleWorkflow`, `SimpleEmailOutbound` / `SimpleEmailTemplate`, `SimpleMessage` (локализация), `SimpleTemplate` / `SimpleTemplatePrinter`, `SimpleWysiwyg`, `SimpleUser` / `SimpleSession` / `SimpleDelegation` / `SimpleUserCriteria`, `SimpleTable` (метаданные), `SimpleSearch`, `SimpleCache`, `SimpleEngine`, `SimpleImport`, `SimpleImage`, `SimpleVcs`, `SimpleMail`, `SimpleLogArchive`, `SimpleRecordDeletionLog`, `SimpleExternalRabbitMQ`; глобалы `current` (текущая запись в BR), `previous` (предыдущее состояние в BR update, read-only), `alert`/`echo`/`print`/`json`/`print_r`/`sleep`/`var_dump` (вывод/отладка).

**Клиентский контекст** (Client Scripts, UI Actions с Client=true): `s_form` (`SimpleForm`), `s_user` (`SimpleUserClient`), `s_list` (`SimpleList`), `s_go` (`SimpleGo`, навигация), `s_modal` (`SimpleModal`), `s_i18n` (`SimpleI18n`), `s_pref` (`SimplePreference`), `SimpleAjax` (вызов серверных скриптов), `SimpleEventBus`, `SimpleRecord` (async, callback-версии), `SimpleStorage`, `SimpleMenu`; глобалы onChange — `fieldName`, `oldValue`, `newValue`.

**Спец-глобалы по контексту:**

| Контекст | Глобалы |
|----------|---------|
| Виджет, server (`server_script.js`) | весь серверный API + `input` (данные от клиента), `data` (данные клиенту) |
| Виджет, client (`client_script.js`) | весь клиентский API + `s_widget` (обмен с сервером), `s_widgets` (межвиджетное) |
| ACL-скрипт | серверный + `current`, `ss`; результат **только** через `answer` (boolean) |
| Notification Script | серверный + `current` (источник), `template` (`SimpleTemplatePrinter`), `email` (`SimpleEmailOutbound`), `event` (`sys_event`) |
| UI Action, server (Client=false) | серверный + `current`, `ss`, `__resolveServerResponse()` (при Wait for server response) |
| Скрипт активности воркфлоу | серверный + `current` (запись процесса), `scratchpad` (JSON, живёт весь контекст), `activity` — запись **`wf_executing_activity`**, не `wf_activity` |

### Запрещённые паттерны (⚠️ КРИТИЧНО)

При наличии платформенного аналога нативные JS/DOM/Node.js и любые методы ServiceNow **запрещены**. Каждый вызов проверять на принадлежность к SimpleOne API.

| Запрещено | Замена | Контекст |
|-----------|--------|----------|
| `GlideRecord`, `GlideElement`, `GlideAggregate`, `GlideDateTime` | `SimpleRecord` / поля `SimpleRecord` / `SimpleDateTime` | Сервер |
| `gs.*` (`gs.info`, `gs.log`, `gs.getUser`, `gs.addInfoMessage`) | `ss.info()` / `ss.log()` / … | Сервер |
| `current.getRefRecord()` | `new SimpleRecord(table).get(current.getValue('ref_field'))` | Сервер |
| `current.getEncodedQuery()`, `setWorkflow()` | нет прямого аналога; формировать вручную / управление через платформу | Сервер |
| `g_form` / `g_user` / `g_list` / `g_navigation` / `g_modal` | `s_form` / `s_user` / `s_list` / `s_go` / `s_modal` | Клиент |
| `s_list.getEncodedQuery()` | `s_list.getQuery()` или фильтр через `&filter=` | Клиент |
| `setReferenceQualifier()` | нет; qualifier задаётся в DSL `dynamic_reference_qualifier_id` (`SysDbColumn`) | Клиент |
| `console.log` | `ss.info()` / `ss.debug()` | Сервер |
| `fetch` / `XMLHttpRequest` | `sws.restRequestV1().execute()`; в виджете — `s_widget.serverUpdate()` | Сервер |
| `new Date()` / `Date.now()` | `new SimpleDateTime()` | Сервер |
| `setTimeout` / `setInterval` | `sleep(seconds)` (сервер) | Сервер |
| `window.location` (сервер) | `ss.setRedirect(...)` | Сервер |
| `window.open` / `window.location.href` | `s_go.open(url)` | Клиент |
| `window.location.reload()` | `s_go.reloadWindow()` | Клиент |
| `window.alert` | `s_form.addInfoMessage()` / `s_form.addErrorMessage()` | Клиент |
| `window.confirm` | модальное окно `s_modal` | Клиент |
| `localStorage` / `sessionStorage` | `SimpleStorage.getItem/setItem` | Клиент |
| `document.cookie` (хранение per-user настроек) | `s_user.getPreference/setPreference` (клиент) / `SimpleUser` (сервер) | Клиент/Сервер |
| `document.getElementById/querySelector` | `s_form.getValue/setValue`; в виджете `s_widget.getElements()` | Клиент |
| `element.style.display`, `innerHTML`, `createElement` | директивы `simple-show` / `simple-if`, шаблон виджета | Виджет |
| HTML строкой с данными (`addTemplate`, шаблонные строки) без экранирования | Данные записи и пользователя (имена, тексты, id) — только через экранирование `&`, `<`, `>`, `"`, `'` перед вставкой в разметку и в атрибуты, включая `event-click="…('${id}')"` | Виджет |

**Динамический HTML в виджете** — проверено на стенде (tfl, `report-trafficlight`, 2026-10-21). Нарушение даёт неработающий интерфейс, который ревью кода не видит:
- `s_widget.addTemplate` санитизирует HTML: вырезает `<input>` (точнее, подменяет своим виджетом), все inline-обработчики `on*`, пользовательские `data-*` (подменяет служебными). Интерактив в такой разметке не живёт.
- `s_widget.setFieldValue` перерисовывает виджет **асинхронно** и пересоздаёт узлы шаблона: содержимое, вставленное в узел шаблона до перерисовки, и навешанные на него обработчики пропадают. Нельзя: `setFieldValue('isShowModal', true)`, затем сразу `container.innerHTML = …` или `addEventListener` на узел внутри модалки.
- Inline `onclick="s_widget_custom.<метод>(event)"` выполняется в глобальной области: метод должен быть объявлен **на верхнем уровне `client()`**, как `s_widget_custom.toggleSelectedObject`. Метод, объявленный во вложенной функции, из inline-обработчика не виден.
- Рабочий паттерн динамического интерактивного списка в виджете — как в боковом окне (`updateAsideModalContent`): разметка через `innerHTML` в контейнер, который `setFieldValue` не перерисовывает, обработчики — на верхнем уровне `client()`. Прежде чем писать новый механизм, найти в этом же виджете рабочий аналог и повторить его.

Запрещены также концепции ServiceNow: `UI Policy` (нет в SimpleOne — логика через Client Scripts), `Script Action`, `Flow Designer`, `GlideAjax`.

Внешняя документация: [Server API](https://docs.simpleone.ru/ru/platform/developer-help/developer-api/server-side-api/) · [Client API](https://docs.simpleone.ru/ru/platform/developer-help/developer-api/client-side-api/) · [Widget API](https://docs.simpleone.ru/ru/platform/developer-help/developer-api/widget-api/api-directives)

### Внешние библиотеки

**Приоритет источников**: 1) SimpleOne API → 2) разрешённый выше vanilla JS → 3) зрелая внешняя библиотека → 4) собственная реализация. Не изобретай велосипед: если существенную неплатформенную логику (сложный парсинг/форматирование, валидация, математика/статистика, работа с датами сверх `SimpleDateTime`, графики и т.п. в виджетах) надёжнее закрыть проверенной библиотекой — предложи её, а не пиши руками.

**Баланс**: не тащи библиотеку ради того, что уже решают платформенный API или базовый vanilla JS (один `map`/`filter`, тривиальная строка) — но и не бойся её там, где она реально упрощает задачу и снижает риск ошибок. Не спамить библиотеками.

**Не применять втихую**: библиотеку нельзя подключить молча. `code-author` показывает 1–3 варианта с названием, лицензией и кратким обоснованием через `NEEDS_USER_INPUT` и ждёт выбора пользователя. До ответа — не писать код, завязанный на библиотеку.

**Лицензии**: предлагать только **пермиссивные** — MIT, ISC, BSD-2/3-Clause, Apache-2.0 (свободно используются в проприетарном продукте). Лицензию проверяй **перед** предложением.

**Два способа подключения** (npm/бандлера/`import()` нет; выбор — по контексту):

- **CDN-загрузка** — клиент виджета (`client_script.js`; доступны браузер/`document`): библиотека грузится с внешнего CDN в рантайме.
- **Вендоринг Script Include** — сервер (Include/BR/Scheduled/`server_script.js`; CDN/DOM нет): код библиотеки хранится в самой сущности и исполняется в серверной песочнице.

**Предложение пользователю** (`NEEDS_USER_INPUT`) — указывать: библиотеку, версию, лицензию, контекст (клиент/сервер) и способ подключения (CDN / вендоринг).

---

## 1. Общие требования

- **ES6+ (⚠️ приоритет 1)**: только `const`/`let` (запрет `var` абсолютный, даже если в проекте есть `var`), arrow functions, template literals, строгое равенство `===`/`!==`.
- **Существующий код как образец (⚠️)**: перед новым скриптом найди 2–3 аналога того же типа в `<product>/domain/business-logic/` (`business-rules/<table>/`, `client-scripts/<table>/`, `ui-actions/<table>/`, `script-includes/`, `widgets/<name>/`). Следуй их стилю, паттернам работы с API, структуре обёрток (⚠️ BR — IIFE, клиентские скрипты — нет).
- **Запросы к БД не в циклах** — один запрос с `addQuery(... 'IN', ids.join(','))`, не N запросов в `for`.
- **`addQuery` > `addEncodedQuery`**: фильтр — через `addQuery('field', 'op', value)`; `addEncodedQuery(str)` только для готовой Encoded-Query-строки (поле-условие, параметр, `s_list.getQuery()`), которую нельзя разобрать на `addQuery`.
- **`get()` не возвращает id**: всегда отдаёт `SimpleRecord` (truthy), поэтому `if (record.get(id))` бесполезен — наличие записи проверять после `get()` по `sys_id`: `record.get(id); if (!record.sys_id) return;`.
- **Dot-walking**: `record.ref.email` (значение связанной записи) — ✅; `record.getValue('field')` — ✅; `record.ref.sys_id` — ❌ (лишний SQL; `getValue('ref')` уже возвращает sys_id).
- **Приоритет low/no-code (⚠️)**: всегда предпочитать декларативные средства (`set_value`, ACL, условия/фильтры, поле `url` в UI Action) написанию скрипта. Скрипт — когда no-code-аналога нет либо он не покрывает логику.
- **Декларативность > скрипты**: если логика выражается условиями (Condition/Filter) или свойствами (`set_value`, `abort_action`) — не писать скрипт.
- **Персональные настройки — User Preference (⚠️)**: per-user значения (выбранное представление, сохранённый фильтр, флаги/состояние UI, последний выбор) хранить через User Preference, **не** в отдельной таблице и **не** в `cookie`/`window`/`localStorage`. Сервер — `new SimpleUser().getPreference(name)` / `.setPreference(name, value)` (синхронно, value ≤255 симв.); клиент — `await s_user.getPreference(name)` / `await s_user.setPreference(name, value)` (async). [Docs](https://docs.simpleone.ru/platform/developer/automation/user-preferences/).
- **Оптимизация**: минимум запросов к БД, без O(n²)+, кэшировать повторно используемые значения.
- **Именование**: ясные имена, неявно указывающие тип/значение (`assignedUserEmail`, `isValidTransition`, не `x`/`res`).
- **Комментарии (⚠️)**: канон — `project-core-rule` («Комментарии»). Кратко: по умолчанию английский (другой язык — по прямой просьбе); только многострочный JSDoc над функцией (`@param {Type} name` / `@returns` — типы из `simpleone-api.d.ts`, их проверяет `tsc` при `checkJs`); комментарии вне JSDoc — исключение для «почему», в формате `//text` без пробела или starred-block; в ACL-скриптах комментарии запрещены, в `.ts` DSL теряются при синхронизации.
- **Обязательные поля**: не генерировать проверки существования для полей с `mandatory: true` — значение гарантировано. Проверка нужна для `mandatory: false`, dot-walking (null), внешних данных (API, параметры).

---

## 2. Клиентская часть

- Избегать запросов к БД на клиенте — получать данные на сервере.
- Использовать Client API (`s_form`, `s_user`, `s_list`) вместо DOM/JS.
- **Навигация/URL**: `s_go.open(url)` (переход/открытие записи), `s_go.reloadWindow()` (перезагрузка), поле `url` в UI Action (без скрипта, предпочтительнее). Фильтр списка — параметр `condition` (Encoded Query), **не** `filter`: `s_go.open('/record/task?condition=state=active')`.

---

## 3. Серверная часть

`SimpleRecord` для работы с записями:

```javascript
const record = new SimpleRecord('incident');
record.addQuery('state', '!=', 'closed');
record.query();
while (record.next()) { /*handle record*/ }
```

Доступные классы — см. серверный контекст в API-справочнике выше.

---

## 4. Виджеты: разметка и стили

> 🎨 Дизайн-токены, типографика, Figma MCP, конвертация макетов — `design-system-rules`. Стандарты виджетов — `widgets`.

- **БЭМ** для CSS-классов: `widget-card`, `widget-card__title`, `widget-card__title--highlighted`.
- **Видимость** — директивы `simple-show="data.x"` / `simple-if="data.x"`, не `style="display:none"`.
- **CSS** — переменные платформы: `var(--primary-background)`, `var(--primary-text)`.

---

## 5. ACL-скрипты

> Полные правила — `access-control`.

Результат — **только** в переменную `answer`; `return` запрещён (ошибка); комментарии запрещены — ни JSDoc, ни `//`.

- ✅ `answer = current.assigned_to == ss.getUserId();`
- ❌ `return current.assigned_to == ss.getUserId();` — `return` в ACL-скрипте недопустим.

---

## 6. Бизнес-правила

- **Декларативные свойства** в `record.ts` вместо скрипта, если логика — присвоение/валидация: `set_value` (JSON присвоения), `abort_action` (отмена транзакции), `add_message` + `message`.
- **`filter_conditions`** — указывать, чтобы правило срабатывало только при изменении ключевых колонок: `'stateISCHANGES^assigned_toISEMPTY'`.
- **`current.update()` запрещён** в BR (рекурсия) — после `setValue` update произойдёт автоматически.
- **Тип правила**: изменение полей `current` → `before`; изменение других записей → `after`; отмена транзакции → `before` (`setAbortAction`). `setAbortAction` запрещён в `async`.

---

## 7. Клиентские скрипты

- **Без IIFE-обёртки** — тело прямо внутри `export default function script() { }` (платформа оборачивает сама). Сравни: BR используют IIFE `(function executeRule(current, previous){...})(current, previous);`.
- **`await` для асинхронных методов** `s_form`: `setValue`, `setVisible`, `setReadOnly`, `setMandatory`, `setSectionDisplay` — требуют `await` (без него UI не обновится). `getValue`, `getUniqueValue` — синхронные.
- **`condition`**: `null` при отсутствии условий (`""` блокирует выполнение скрипта). Избегать `condition` вообще — он вызывает серверную проверку; фильтровать на клиенте (`if (s_form.getValue('state')==='draft') return;`). Исключение — неизбежный запрос на сервер (dot-walking).
- **Типы**: `onLoad` (`s_form`, `s_user`), `onChange` (+ `fieldName`/`oldValue`/`newValue`), `onSubmit`, `onCellEdit` (`s_list`).

### Управление видимостью/обязательностью (⚠️ КРИТИЧНО)

1. **Двусторонняя обработка** — обрабатывать ВСЕ ветки условия, включая возврат к default. Нельзя только показывать поле, забыв скрыть обратно.
2. **При скрытии — полный сброс**: `setMandatory(field,false)` (иначе форму не сохранить) → `clearValue(field)` (если логика требует) → `setVisible(field,false)`.
3. **`onLoad` + `onChange` парой**: если поле скрыто по умолчанию и появляется по условию — нужен `onLoad` (начальное состояние) И `onChange` (реакция). Без `onLoad` поле будет видно при загрузке до первого изменения.
4. **Несколько зависимых полей** от одного — группировать в одном `onChange`.

```javascript
export default function script() {
  const showAggregate = s_form.getValue('quality_indicator_type') === 'quantity';
  await s_form.setVisible('aggregate_data', showAggregate);
  await s_form.setMandatory('aggregate_data', showAggregate);
  if (!showAggregate) await s_form.setValue('aggregate_data', '');
}
```

---

## 8. UI-действия

- Перенаправление — поле `URL`, не скрипт.
- Контекст: `Client` включён → клиент (`s_form`, `s_user`); выключен → сервер (`current`, `ss`).
- При `Wait for server response` — вызвать `__resolveServerResponse()` для снятия блокировки кнопок.

---

## 9. Скрипты уведомлений

IIFE с 4 параметрами; HTML-шаблоны используют `${current.field}` для подстановки. Детали — `notifications`.

Параметры: `current` — запись-источник; `template` — `SimpleTemplatePrinter`; `email` — `SimpleEmailOutbound` (`addAddress`/`setSubject`/`setBody`); `event` — `sys_event`.

```javascript
export default function script() {
  (function runNotificationScript(current, template, email, event) {
    const user = new SimpleRecord('sys_user');
    user.get(current.getValue('assigned_to'));
    email.addAddress(user.getValue('email'));
  })(current, template, email, event);
}
```

---

## 10. Скрипты активностей воркфлоу

Серверный контекст. Детали по типам активностей — `workflows`.

`activity` — это `wf_executing_activity` (запись о выполнении блока), а не настройка активности. `scratchpad` переносит значения между блоками одного контекста: `Record Generator` не возвращает ID созданной записи, поэтому её ищут по полю `wf_executing_activity`.

```javascript
export default function script() {
  scratchpad.generatorActivityId = activity.sys_id;
}

export default function script() {
  const task = new SimpleRecord('task');
  task.get('wf_executing_activity', scratchpad.generatorActivityId);
  if (task.sys_id) {
    task.state = '3';
    task.update();
  }
}
```

---

## 11. Скрипты виджетов

Глобалы — см. спец-таблицу в API-справочнике. Детали — `widgets`.

`server_script.js` — серверный контекст: `input`, `data` + серверный API; **НЕ** `s_widget`/`s_form`/`current`. `client_script.js` — клиентский контекст: `s_widget`, `s_widgets` + клиентский API; рендер — через `s_widget.getElements()`.

```javascript
export default function server() {
  (function() {
    if (!input.record_id) return;
    const record = new SimpleRecord('demo_story');
    record.get(input.record_id);
    data.title = record.getDisplayValue('subject');
    data.state = record.getValue('demo_stage');
  })();
}

export default function client() {
  (async () => {
    const recordId = s_form.getUniqueValue();
    if (!recordId) return;
    s_widget.setFieldValue('record_id', recordId);
    await s_widget.serverUpdate();
    const title = s_widget.getFieldValue('title');
  })();
}
```
