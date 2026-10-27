---
description: Стандарты воркфлоу — Workflow, версия, активности, выходы, переходы. Загружать при работе с workflows/**.
paths: '**/workflows/**/*'
user-invocable: false
---

# Воркфлоу (Workflow, Version, Activities, Exits, Transitions)

DSL-экспорт, TypeScript. Классы: `WfWorkflow`, `WfWorkflowVersion`, `WfActivity` и его наследники, `WfActivityExit`, `WfTransition`. `sys_id` — только в экспорте, как у любой записи; у записей графа (версия, активности, выходы, переходы) он меняется после каждого деплоя, и SimpleGen записывает новые значения обратно. [Docs](https://docs.simpleone.ru/platform/developer/business-logic/workflows/).

## Модель (⚠️ КРИТИЧНО)

Воркфлоу — **одна VCS-запись** `wf_workflow`, внутри которой платформа собирает весь граф при публикации. Дочерние таблицы своих VCS-записей не имеют.

```
WfWorkflow → WfWorkflowVersion (workflow_id)
           → WfActivity* (workflow_version_id, activity_type_id)
             → WfActivityExit (activity_id, activity_type_exit_id)
               → WfTransition (activity_exit_id → activity_to_id)
```

Уникальность: workflow — `name`+`table_id`; активность — `name` в рамках версии; выход — `activity_type_exit_id` в рамках активности; переход — пара выход→активность.

## Что создаёт платформа сама (⚠️ КРИТИЧНО)

Это главное отличие от форм: **большая часть графа появляется без тебя**.

| Действие | Что платформа создаёт сама |
|----------|----------------------------|
| Создание `WfWorkflow` | версию (checked out), активности `Begin` (x=300, y=100) и `End` (x=600, y=600), выход `Always` у Begin и **переход Begin.Always → End** |
| Создание любой активности | все её выходы из `wf_activity_type_exit` соответствующего типа (у `Switch` — только `Default`, в том числе при `type: "field"`: выходы по опциям колонки платформа не создаёт) |
| Создание `Timer` без скрипта | скрипт по умолчанию `answer = 0;` с комментарием — после sync он появится в `.js`-файле |
| Публикация версии | новую VCS-запись воркфлоу в **текущем локальном паке** |
| Check out | полную копию версии: новые `sys_id` у всех активностей, выходов и переходов |
| Откат пака (VCS rollback) | удаляет только строку `wf_workflow`: версии с графом остаются без родителя |

Отсюда правила:
- **Папка воркфлоу — единица деплоя.** Любое изменение в любом её файле означает: SimpleGen делает check out, **полностью очищает** рабочую копию и создаёт граф заново из всех файлов папки, затем публикует. Поэтому папка всегда описывает граф целиком: версию, `Begin`, `End`, все активности, все выходы, все переходы. Что в папке — то и на стенде; чего в папке нет, после деплоя не будет.
- **Воркфлоу описывай целиком**, включая версию, `Begin`, `End` и все выходы — иначе переходам не на что ссылаться. Записи, которые платформа создаёт сама (версия, `Begin`/`End`, выходы типов, переход `Begin → End`), SimpleGen привяжет к записям DSL. Дублей не будет.
- **Выходов не больше, чем у типа** (таблица каталога ниже), с тем же `activity_type_exit_id` и `name`: лишний выход станет второй веткой. Исключение — `Switch`: у него один выход `Default` из типа, остальные создаются вручную.
- Опубликованная версия **не редактируется**: платформа запрещает update и delete, но разрешает insert. Любая запись, отправленная в опубликованную версию, останется там навсегда. SimpleGen сам делает `check out → очистка → граф из DSL → publish`; руками на стенде не трогать.
- После каждого деплоя SimpleGen записывает новые `sys_id` всех записей папки обратно в DSL; этот коммит повторного деплоя не вызывает. Ссылки между записями графа всегда разрешаются по именам экспортов, а не по `sys_id`.
- Переименование активности — это удаление и создание заново (id у неё и так новый после каждой публикации). Запущенные экземпляры процесса продолжают работать на своей версии.
- Если SimpleGen отказался публиковать версию (см. инварианты ниже), рабочая копия уходит в архив, опубликованная версия продолжает работать. Исправь DSL и запусти снова — будет новый check out от опубликованной версии.
- Удалить воркфлоу — удалить его папку. SimpleGen удаляет его вместе со всеми версиями через редактор; Table API такое удаление запрещает.
- Если воркфлоу пропал со стенда (откатили пак, удалили руками), следующая сборка создаст его заново из DSL, а оставшиеся от него версии подвесит к новому как архивные.

## Размещение и поля

Всё — в `workflows/{workflow.table_id.name}/{kebab(workflow.name)}/`. Константа — `toCamelCase(name, 50)`.

| Тип | Файл | Ключевые поля |
|-----|------|---------------|
| `WfWorkflow` | `workflow.ts` | `name`✱, `table_id`✱, `active`, `condition_type`, `condition` |
| `WfWorkflowVersion` | `workflow.ts` | `workflow_id`✱, `published`, `checked_out` (управляет платформа) |
| `WfActivity` и наследники | `activities.ts` | `workflow_version_id`✱, `activity_type_id`✱, `name`, `x`✱, `y`✱ + поля своего типа |
| `WfActivityExit` | `exits.ts` | `activity_id`✱, `activity_type_exit_id`, `name`, `order`, `exit_condition`, `condition`, `script` |
| `WfTransition` | `transitions.ts` | `activity_exit_id`✱, `activity_to_id`✱ |

**`condition_type`** (когда стартует процесс): `manual` — вручную или `new SimpleWorkflow(id).start(current)`; `auto` — по условию из `condition`; `rem_driven` — для записей с REM-моделью.

**Форматы полей** (проверено на стенде; другое значение платформа отвергнет или молча исказит):

| Поле | Формат |
|------|--------|
| `duration` у `Timer` | целое число **миллисекунд** строкой: `"300000"` = 5 минут. Текст вида `"5minutes"` отвергается |
| `set_values` у `Set Values` и `Record Generator` | JSON-строка вида `"{\"1\": {\"state\": \"3\"}}"` — номер строки → `{колонка: значение}`. Короткую форму `{"state": "3"}` платформа сама приведёт к этому виду; строку `state=3` не принимает |
| `condition`, `completion_conditions`, `run_conditions` | закодированный запрос: `priority=1`, `state=7` |
| `employees`, `groups`, `roles` у `User-approval` | массив: `["155931135900000001"]` или `[admin]` (импорт роли) |
| `task_type` у `Record Generator` | ссылка на таблицу-наследник `task` |
| `event_id` у `Create Event` | импорт из `platform/other/event-registers.ts`; `param_1`…`param_5` — строки, строковое значение в кавычках: `"'rejected'"` |
| `workflow_id` у `Subflow` | импорт воркфлоу из соседней папки; можно создавать в одном патче с ним |
| `type` у `Switch` | `custom` (выходы вручную), `field` (+ `column_id`), `attribute` (+ `re_attribute_id`) |

**Ручные выходы `Switch`**: обязательно `table_id` (таблица воркфлоу), `condition`, `name`; `activity_type_exit_id: null`. `order` и `exit_condition` платформа проставляет сама (0, 1, …) — заданные значения она перезапишет. `activity_id` выхода после создания менять нельзя (read-only) — чтобы «переподвесить» выход, удали его и создай новый.

**Файлы скриптов** называются по активности: `script` → `<kebab(name)>.js`, `script_approvers` → `<kebab(name)>-approvers.js`, `appr_script` → `-approved.js`, `rej_script` → `-rejected.js`, `non_mandatory_script_approvers` → `-non-mandatory-approvers.js`. Если назвать иначе, sync переименует файл.

**Координаты `x`/`y` обязательны** — без них все блоки лягут в одну точку и схему невозможно будет читать. Сетка: `Begin` (300, 100), далее шаг 160 вниз, ветвления ±300 по x, `End` внизу.

**Имена констант**: активность — `toCamelCase(name)`; выход — `<активность><Выход>` (`beginAlways`); переход — `<выход>To<целевая активность>` (`beginAlwaysToUserApproval`), потому что у перехода нет своего имени.

## Каталог типов активностей

`activity_type_id` — ссылка на запись `wf_activity_type`; брать **только** именованным импортом, ID не выдумывать. Каталог лежит по приложению записи: типы платформы — в `platform/other/workflow-activity-types.ts`, а `Switch` принадлежит приложению Simple и лежит в `sa/domain/other/workflow-activity-types.ts` (выход `Default` — рядом, в `workflow-activity-type-exits.ts`). Класс активности обязан соответствовать типу.

| Тип | Класс DSL | Выходы типа (order) |
|-----|-----------|---------------------|
| Begin | `WfActivity` | Always (1) |
| End | `WfActivity` | — |
| Run Script | `WfActivityRunScript` | Always (1) |
| Set Values | `WfActivitySetValues` | Always (10) |
| Join | `WfActivityJoin` | Complete (10), Incomplete (20) |
| Record Generator | `WfActivityRecordGenerator` | Always (10) |
| Create Event | `WfActivityCreateEvent` | Always (10) |
| If | `WfActivityIf` | Yes (10), No (20) |
| Switch | `WfActivitySwitch` | Default (999) + созданные вручную |
| Wait Condition | `WfActivityWaitCondition` | Always (10) |
| User-approval | `WfActivityApproval` | Approved (10), Rejected (20) |
| Subflow | `WfActivitySubflow` | Always |
| Timer | `WfActivityTimer` | Always |

`Run Script` и `Set Values` выполняются сразу (`When to run = Immediately`), остальные — по изменению записи. `Subflow` ссылается только на **опубликованный** воркфлоу.

## Инварианты графа (⚠️ КРИТИЧНО)

- У каждой активности, кроме `End`, должен быть **хотя бы один выход**, иначе платформа не даст опубликовать версию.
- У каждого выхода должен быть переход — иначе процесс, уйдя в эту ветку, встанет навсегда.
- `activity_type_id` обязателен. Активность, сохранённая без типа, ломает воркфлоу необратимо: платформа её не удалит и не даст сделать check out.
- Нельзя вести **два выхода одной активности** в один `Join` — Join всегда выберет `Incomplete`.
- Переход соединяет выход и активность **одной и той же версии**. Платформа это не проверяет — проверяет SimpleGen перед публикацией, как и дубли (один выход дважды в одну активность).

Нарушение последних двух SimpleGen ловит перед публикацией и публиковать отказывается: рабочая копия уходит в архив, опубликованная продолжает работать. Исправь DSL и запусти сборку снова.

## Скрипты

Скриптовые поля выносятся в `.js` рядом с DSL-файлом (см. `script-rules`): `script` у `Run Script` / `If` / `Wait Condition` / `Timer` / `Record Generator` / выхода; `appr_script`, `rej_script`, `script_approvers`, `non_mandatory_script_approvers` у `User-approval`.

В скриптах воркфлоу доступны `current`, `scratchpad` (JSON, переживает шаги контекста) и `activity` — запись `wf_executing_activity`, а **не** `wf_activity`. Скрипты участников согласования и `Timer` возвращают результат через `answer` (список id пользователей или число секунд); скрипт `Record Generator` заполняет `new_task`. В `Run Script` не вызывай `current.update()` — запись сохранит сама платформа.

## Пример — новый воркфлоу

Проверен деплоем на стенде: «Begin → If → Run Script → End», ветка «No» сразу в End.

```typescript
// workflow.ts
export const simplegenDemo: WfWorkflow = new WfWorkflow({
    "name": "SimpleGen demo", "table_id": task, "active": false, "condition_type": "manual",
});
export const simplegenDemoVersion: WfWorkflowVersion = new WfWorkflowVersion({
    "name": "SimpleGen demo", "table_id": task, "workflow_id": simplegenDemo,
});

// activities.ts — Begin и End тоже описаны: их создаст платформа, SimpleGen привяжет
import { If as ifType, begin as beginType, end as endType, runScript as runScriptType } from "../../../../../platform/other/workflow-activity-types";
import markUrgentScript from "./mark-urgent.js";
export const begin: WfActivity = new WfActivity({
    "activity_type_id": beginType, "name": "Begin", "workflow_version_id": simplegenDemoVersion, "x": 300, "y": 100,
});
export const isHighPriority: WfActivityIf = new WfActivityIf({
    "activity_type_id": ifType, "name": "Is high priority", "condition": "priority=1", "table_id": task,
    "workflow_version_id": simplegenDemoVersion, "x": 300, "y": 260,
});
export const markUrgent: WfActivityRunScript = new WfActivityRunScript({
    "activity_type_id": runScriptType, "name": "Mark urgent", "script": markUrgentScript as unknown as string,
    "workflow_version_id": simplegenDemoVersion, "x": 600, "y": 420,
});
export const end: WfActivity = new WfActivity({
    "activity_type_id": endType, "name": "End", "workflow_version_id": simplegenDemoVersion, "x": 300, "y": 580,
});

// exits.ts — ровно выходы типов, с их activity_type_exit_id
import { always as beginAlwaysType, always2 as runScriptAlwaysType, no as ifNoType, yes as ifYesType } from "../../../../../platform/other/workflow-activity-type-exits";
export const beginAlways: WfActivityExit = new WfActivityExit({
    "activity_id": begin, "activity_type_exit_id": beginAlwaysType, "name": "Always", "order": 1, "exit_condition": "true",
});
export const isHighPriorityYes: WfActivityExit = new WfActivityExit({
    "activity_id": isHighPriority, "activity_type_exit_id": ifYesType, "name": "Yes", "order": 10,
    "exit_condition": "activity.result == 'true'",
});
// …isHighPriorityNo, markUrgentAlways — так же

// transitions.ts — у каждого выхода ровно один переход
export const beginAlwaysToIsHighPriority: WfTransition = new WfTransition({
    "activity_exit_id": beginAlways, "activity_to_id": isHighPriority,
});
export const isHighPriorityYesToMarkUrgent: WfTransition = new WfTransition({
    "activity_exit_id": isHighPriorityYes, "activity_to_id": markUrgent,
});
// …isHighPriorityNoToEnd, markUrgentAlwaysToEnd
```

Имена импортов из каталога (`always2`, `If`) бери из выгруженных `platform/other/workflow-activity-type*.ts` своего проекта: выход Always есть у многих типов, поэтому экспорты пронумерованы. Нужный выход определяй по его `activity_type_id`, не угадывай.
