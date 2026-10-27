---
description: Стандарты структур данных — таблицы, колонки, choice-опции, state-модели и переходы, нумерация записей, переопределения колонок. Загружать при работе с data-structure/**.
paths: '**/data-structure/**/*'
user-invocable: false
---

# Структуры данных

DSL-экспорт метаданных SimpleOne: TypeScript (`.ts`), экспорт типизированных констант. Классы: `SysDbTable`, `SysDbColumn`, `SysChoice`, `SysStateModel`, `SysStateTransition`, `SysNumber`, `SysFilterOptionDynamic`. Описаны только архитектурно значимые поля; остальные — значения по умолчанию (`false`/`""`/`0`/`null`). `sys_id` — только в экспорте существующих записей; при создании новой сущности **убрать** (см. `project-core-rule`).

Docs: [Tables](https://docs.simpleone.ru/platform/admin-help/data-structure/tables/) · [Columns](https://docs.simpleone.ru/platform/developer-help/data-structure/columns-in-data-structure/) · [Choices](https://docs.simpleone.ru/platform/admin-help/data-structure/columns/column-types/choice-fields) · [State Models](https://docs.simpleone.ru/ru/esm/auxiliary-tools/state-models) · [Record Numbers](https://docs.simpleone.ru/platform/admin-help/data-structure/record-numbers) · [Dynamic Filters](https://docs.simpleone.ru/platform/admin-help/auxiliary-elements/condition-builder/filters#dynamic-filters). Альтернатива наследованию — REM (`record-extension-model`).

## Наследование (`parent_id`)

Дочерняя таблица наследует **все** колонки родителя (и всей цепочки). Колонки родителя НЕ дублируются в файле дочерней таблицы. Перед созданием колонки — проверить, нет ли одноимённого `column_name` у родителя/предков (иначе ошибка уникальности при деплое): открыть `tables/<table>.ts` → найти `parent_id` → проверить `columns/<parent>.ts` (в `sa/` или `platform/`) по всей цепочке. Типичные цепочки: `itsm_incident → task → sys_metadata`.

## Уникальность (нарушение → ошибка деплоя)

| Сущность | Уникальный ключ |
|----------|------------------|
| `sys_db_table` | `name` (глобально) |
| `sys_db_column` | `column_name` + `table_id` (включая унаследованные!) |
| `sys_choice` | `table_id` + `column_id` + `value` + `language` |
| `sys_state_model` | `table` + `name` |
| `sys_state_transition` | `state_model` + `state` |
| `sys_number` | `table_id` + `column_id` (префикс — желательно уникальный) |
| `sys_filter_option_dynamic` | `title` + `referenced_table_id` |

## Именование колонок (`column_name`)

Только латиница, snake_case, строчные. Продуктовый префикс — из `AGENTS.md` (`pda_`/`itsm_`/`crm_`). Суффиксы по типу:

| Тип | Суффикс/префикс | Примеры |
|-----|-----------------|---------|
| Reference | `_id` | `assigned_to_id`, `category_id` |
| Boolean | `is_`/`has_` | `is_active`, `has_children` |
| Date/DateTime | `_at`/`_date` | `created_at`, `due_date` |
| Duration | `_duration` | `resolution_duration` |
| List | `_ids`/`_list` | `member_ids`, `role_list` |
| Conditions | `_condition`/`_filter` | `filter_condition` |

## Обязательность (`mandatory`)

Принцип: «работает ли функциональность без поля?». Нет → `true`; да → `false`. Функциональные (`state`, `priority`, `assigned_to`) и идентификационные (`number`, уникальное `name`) → `true`. Информационные (`description`, `comments`, `notes`) → `false`. Поле в условиях/вычислениях BR или валидации CS → скорее `true`; только для отображения → `false`. В спецификации обязательность указывать явно и обосновывать.

## Зарезервированные имена колонок (⚠️ КРИТИЧНО)

Создание колонки с этими именами → ошибка «This name is reserved by system». Платформа создаёт их автоматически: `number` (при `SysNumber`), `sys_id`, `sys_created_at/by`, `sys_updated_at/by` (всегда); `sys_db_table_id`, `application_id`, `policy` (появляются в DSL только после `init`/`sync` — вручную не объявлять). Для нумерации — настроить `SysNumber`, привяжется к авто-колонке `number`.

## Размещение и именование сущностей

Имя константы для всех типов: `toCamelCase(display_name || name || title, 50)`, fallback `<type>_${sys_id}`. Группировка и пути:

| Тип (класс) | Путь | Группировка |
|-------------|------|-------------|
| Tables (`SysDbTable`) | `{app}/data-structure/tables/{name}.ts` | таблица в отдельном файле |
| Columns (`SysDbColumn`) | `{colFolder}/data-structure/columns/{table.name}.ts` | по `table_id`; `colFolder` — каталог приложения-владельца таблицы (`platform` для платформенных `sys_*`, `sa/domain` для SA-таблиц, иначе `{app}`) |
| Choices (`SysChoice`) | `{app}/data-structure/choice-options/{table.name}.ts` | по `table_id` |
| State Models (`SysStateModel`) | `{app}/data-structure/state-models/{table.name}.ts` | по `table` |
| State Transitions (`SysStateTransition`) | `{app}/data-structure/state-transitions/{state_model.table.name}.ts` | по таблице модели |
| Record Numbers (`SysNumber`) | `{app}/data-structure/record-numbers.ts` | все в одном файле |
| Dynamic Filters (`SysFilterOptionDynamic`) | `{app}/data-structure/dynamic-filter-options/{referenced_table.name}/records.ts` | по `referenced_table_id` (см. `dynamic-filter`) |

`platform/` — read-only; `sa/` — read-only без вендора, иначе `sa/domain/`; `<product>/` — редактируемый.

## Префикс `c_`

Не обязателен и не добавляется автоматически — зависит от соглашений проекта, уточнять до генерации (детали: `project-core-rule`). Если используется — полное имя с `c_` совпадает в `SysDbTable.name`, скриптах (`new SimpleRecord('c_...')`) и Encoded Query. После создания таблицы — `simplegen.product_sync`.

## Префикс `sys_`

`sys_` — признак принадлежности платформе или SA, а не запрещённый префикс. В продуктовом приложении новые таблицы называть с продуктовым префиксом (`pda_`/`itsm_`/`crm_`, опц. `c_`). В `sa/` с вендорскими правами создание `sys_*` **допустимо** — сущности ложатся в `sa/domain/`. Действуют обычные ограничения: `name` уникален глобально (проверить `platform/` и `sa/`), `platform/` не редактировать, зарезервированные имена колонок — ниже.

## Ключевые свойства колонок

- **`display_by_ref: true`** — поле, отображаемое при ссылке на запись этой таблицы (иначе показывается `sys_id`). Ставить на `name`/`title`; ровно **одно** поле таблицы.
- **`dependent_on_column_id`** — зависимость от другой колонки той же таблицы. **Обязательно для типа `Conditions`** (без него конструктор условий крашит форму). Также для зависимых Choice-полей.
- **`read_only: true`** — колонка только для чтения в UI: значение нельзя ввести/изменить на форме. Скрипты не должны присваивать ей значение (`setValue` на read_only-колонку бессмысленно). Требование «поле нельзя изменять» (совсем, после создания, всем кроме роли) закрывается **write-ACL на колонку**, а не этим чекбоксом — см. «Нередактируемое поле: чем закрывать» ниже.
- **`unique: true`** — значение колонки уникально в пределах таблицы (валидация на уровне БД); дубликат → ошибка при сохранении.
- **`default_value`** — значение по умолчанию при создании записи. Статическое (`"active"`, `"0"`) либо динамическое — префикс `javascript:` и результат через `setResult(...)`: `"javascript: setResult(ss.getUserID());"`, `"javascript: setResult('');"` (пустое поле).

### Нередактируемое поле: чем закрывать (⚠️ КРИТИЧНО)

Чекбокс `read_only` — самый узкий из механизмов, он закрывает поле **для всех и навсегда**. Выбор:

| Требование | Механизм | Почему не чекбокс |
| ---------- | -------- | ----------------- |
| Значение не меняют ни пользователь, ни скрипт (константа, вычисляется платформой) | `read_only: true` | — |
| Поле заполняется автоматически (Business Rule, `set_value`, интеграция), руками не редактируется | **write-ACL** на колонку (`operation: "write"`, `column_id`, `answer = false` или условие) | Чекбокс заблокирует и автозаполнение — значение перестанет записываться |
| Поле недоступно части ролей | **write-ACL** с `role_ids` / условием | Чекбокс не различает роли |
| Поле закрывается по статусу или другому условию записи | **write-ACL** с `condition`/`script` (+ клиентский скрипт для UI-обратной связи) | Чекбокс не знает состояния записи |
| Поле должно выглядеть заблокированным на форме при определённых значениях | Клиентский скрипт `s_form.setReadOnly(...)` (`await`) | Чекбокс статичен, реакции на изменение полей формы нет |

Правило: серверный запрет — write-ACL (`access-control` → «Запрет изменения поля»), поведение формы — клиентский скрипт (`business-logic`, `script-rules`). Требование «поле не редактируется вручную» из спеки **не** переводится в чекбокс по умолчанию — выбор фиксируется в разделе «Предварительное техническое решение» таск-плана.

### Чек-лист по типам колонок (⚠️ КРИТИЧНО)

| Тип | Обязательно | Опционально | Частая ошибка |
|-----|-------------|-------------|----------------|
| Reference (1 ссылка) | `reference_id`, `on_delete` | `display_by_ref` (отображение на форме), `dynamic_reference_qualifier_id` (фильтр по контексту), `dependent_on_column` | без `display_by_ref` → виден `sys_id`; без qualifier → все записи |
| List (мн. ссылок) | `reference_id`, `on_delete` | `display_by_ref`, `dynamic_reference_qualifier_id` | путают с Reference; без `reference_id` → не знает таблицу элементов |
| Conditions (encoded query) | `dependent_on_column` (на колонку типа Reference/Table Name) | `dynamic_reference_qualifier_id` | без `dependent_on_column` → конструктор не показывает поля |

`dynamic_reference_qualifier_id` (`SysFilterOptionDynamic`) должен быть создан **до** колонки (см. порядок зависимостей (`project-core-rule`)).

## Choice: value vs title (⚠️ КРИТИЧНО)

| Поле | Назначение | Формат |
|------|------------|--------|
| `value` | database value (в БД, в скриптах) | `lowercase_snake_case`, только латиница: `active`, `in_progress` |
| `title` | display value (пользователю) | человекочитаемый, любой язык: `Active`, `В работе` |

В скриптах ВСЕГДА `value`, никогда `title`: `current.setValue('state', 'active')` ✅; `setValue('state', 'Активный'/'Active')` ❌.

## Choice: «не выбрано» — свойство колонки, не опция

❌ **Никогда** не создавать `SysChoice` вида `value: "none"` / `"empty"` / `"not_selected"` / `"—"`. Это реальное значение в БД: попадёт в Encoded Query, в отчёты, в условия BR, и «пустым» полем не является.

✅ Пустое значение включается на **колонке**:

| Свойство колонки | Значение | Зачем |
|------------------|----------|-------|
| `choice_type` | `"1"` | Dropdown with -- None --: платформа сама добавляет пункт «пусто» |
| `default_value` | `"javascript: setResult('');"` | запись открывается без выбранной опции (иначе платформа подставит первую) |

## Примеры DSL

```typescript
// tables/pda_project.ts
import { SysDbTable } from "../../../../platform/data-structure/columns/sys_db_table";
import { task } from "../../../../sa/domain/data-structure/tables/task";
export const pdaProject: SysDbTable = new SysDbTable({
    "name": "pda_project", "parent_id": task, "title": "Project", "is_extendable": true,
});

// columns/pda_project.ts — String + Reference с динамическим фильтром
import { SysDbColumn } from "../../../../platform/data-structure/columns/sys_db_column";
import { sysUser } from "../../../../platform/data-structure/tables/sys_user";
import { me } from "../dynamic-filter-options/sys_user/records";
export const pdaName: SysDbColumn = new SysDbColumn({
    "column_name": "pda_name", "table_id": pdaProject, "column_type_id": "String",
    "max_length": 255, "mandatory": true, "display_by_ref": true, "title": "Name",
});
export const pdaOwner: SysDbColumn = new SysDbColumn({
    "column_name": "pda_owner", "table_id": pdaProject, "column_type_id": "Reference",
    "reference_id": sysUser,                      // целевая таблица обязательна
    "dynamic_reference_qualifier_id": me,         // SysFilterOptionDynamic
    "mandatory": true, "title": "Owner",
});

export const pdaStatus: SysDbColumn = new SysDbColumn({
    "column_name": "pda_status", "table_id": pdaProject, "column_type_id": "Choice",
    "choice_type": "1",                                // Dropdown with -- None --
    "decoration": "radio",                            // радиокнопки на форме
    "default_value": "javascript: setResult('');",     // старт без выбранной опции
    "mandatory": false, "title": "Status",
});

// choice-options/pda_project.ts — только реальные значения, без опции «не выбрано»
import { SysChoice } from "../../../../platform/data-structure/columns/sys_choice";
import { pdaStatus } from "../columns/pda_project";
export const statusActive: SysChoice = new SysChoice({
    "table_id": pdaProject, "column_id": pdaStatus,
    "value": "active", "title": "Active", "order": 100,   // value=database, title=display
});

// state-models/pda_project.ts
export const pdaProjectStateModel: SysStateModel = new SysStateModel({
    "table": pdaProject, "name": "Project State Model", "active": true,
});

// state-transitions/pda_project.ts
export const activeState: SysStateTransition = new SysStateTransition({
    "state": statusActive, "state_model": pdaProjectStateModel,
    "available_transitions": [statusCompleted], "autotransition": false,
    "policy": "Protected", "record_activity": "-1",
});

// record-numbers.ts
import { number } from "./columns/pda_project";
export const pdaProjectNumber: SysNumber = new SysNumber({
    "table_id": pdaProject, "column_id": number, "prefix": "PRJ", "number": 1000, "number_of_digits": 7,
});
```
