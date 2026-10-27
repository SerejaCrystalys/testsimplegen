---
description: Базовые правила проекта SimpleOne DSL — поведение агента (не выдумывать ID/методы/имена), приоритет источников, контекст AGENTS.md, структура platform/·sa/·domain/·tasks/, sys_id, порядок деплоя и запрет менять стенд через API без прямой просьбы пользователя. Базовый контекст для любой работы с DSL.
user-invocable: false
---
# Правила проекта SimpleOne DSL

## Поведение ИИ (⚠️ КРИТИЧНО)

- **НЕ выдумывай и НЕ предполагай** значения полей, ID, имена таблиц/колонок, API-методы, бизнес-логику, переводы, значения фильтров. Если данных нет в контексте проекта (`platform/`, `sa/`, `<product>/domain/`) — задай уточняющий вопрос или явно укажи `[УТОЧНИТЬ]`.
- API — **только** из справочника SimpleOne; сверяться по индексу сигнатур `api-index/` (порядок чтения — `script-rules`). Метода нет в справочнике → он не существует. SimpleOne — **не** ServiceNow.

## Контекст

- Корневой `AGENTS.md` — общий контекст проекта и список команд simplegen. `AGENTS.md` каждого модуля — описание модуля; добавляй в контекст AGENTS.md затрагиваемых модулей.
- Контекст текущей фичи: `SIMPLEGEN_ACTIVE_FEATURE` в `.env` (формат `<product>/<task_id>`). Отсюда выводится путь задачи `<current_feature_path>` = `tasks/<task_id>` — именно он подставляется в пути `<product>/<current_feature_path>/…` в остальных скиллах.

## Приоритет правил (⚠️ КРИТИЧНО)

При конфликте источников (от высшего к низшему): **1.** явные правила из `.claude/skills/` (именование, API, стиль, ES6+) → **2.** `AGENTS.md` продукта → **3.** существующий код проекта → **4.** общие практики.

Правило «следуй существующему коду» (3) **НЕ отменяет** правила из `.claude/skills/` (1). Если код использует `var`/`function`, а правила требуют `const/let` и arrow functions — применять правила.

## Структура

- `platform/` — read-only, никогда не редактировать.
- `sa/` — read-only без вендорских прав; с вендорскими правами — полноценный продукт (`domain/`, `tasks/`, `simplegen.json`).
- `<product>/domain/` — DSL бизнес-решения (генерируется/редактируется; для SA с вендором — `sa/domain/`).
- `<product>/tasks/` — артефакты задач (`_spec.md`, `_task_plan.md`, `simplegen.json`, `<task_id>.sop`, `build_logs/`).
- `platform/` и `sa/` обновляются через `simplegen.init` / `simplegen.product_sync`.

## Код и файлы

- Следуй DSL-спецификациям правил сущностей в `.claude/skills/` (навигация — `rule-router`).
- Не создавай `AGENTS.md` и документацию (`*.md`) без явного запроса.

## Комментарии

**1. Язык — по умолчанию английский.** Комментарии и JSDoc пишутся по-английски, даже когда спека, отчёты агентов и интерфейсные тексты — на русском. Кириллица в комментарии без явной просьбы — замечание ревью (`AUTO`: перевести). Если пользователь прямо попросил другой язык (постановка задачи, `AGENTS.md`, сообщение в чате) — писать на нём, это не нарушение. Это не касается локализуемых строк и текстов в `SourceMessage`/`sys_translation` — там язык определяет `interface-text-rules`.

**2. Формат — только JSDoc.** Единственная разрешённая форма комментария — блок JSDoc над функцией/методом/классом:

**3. Комментарии вне JSDoc — очень нежелательны.** Допустимы как исключение и только там, где код не может объяснить себя сам: обход платформенного ограничения, неочевидный обязательный порядок вызовов, значение/формат, навязанный внешней системой. Объясняют **почему**, никогда **что**. Если тянет пояснить *что* делает код — вместо комментария переименовать переменную или вынести функцию с говорящим именем.

Формат исключения задан линтером: однострочный — `//text` (без пробела после слэшей), многострочный — только starred-block. `// text`, `/* text */` и два `//` подряд — ошибки ESLint (`spaced-comment`, `multiline-comment-style: starred-block`).

## Версионирование

Перед любой командой simplegen (кроме `simplegen.init`): сравни `version` из `.simplegen/state.json` с выводом `simplegen --version`.
- CLI ниже → обновить: `npm update @simplegen/simplegen`
- CLI выше → проект инициализирован старой версией: `/simplegen.init`
- Совпадают → продолжить

`initial_commit_sha` фичи фиксируется при первом `simplegen.code_generate`; diff `initial_commit_sha..HEAD` показывает объём изменений фичи.

## sys_id (⚠️ КРИТИЧНО)

- **Новая сущность**: поле `sys_id` должно **полностью отсутствовать** в объекте. Не `""`, не `null`, не значение — а отсутствовать. ID генерирует платформа при импорте; любое указание `sys_id` ломает парсинг/деплой.
- **Существующая запись (DSL-экспорт)**: использовать существующий `sys_id` — он идентифицирует запись.

## Поля, которыми управляет платформа

Эти поля встречаются в выгруженном DSL (после `init`/`sync`), но **при деплое вырезаются из payload** — их значение в DSL ни на что не влияет. Не задавать вручную у новых записей и не «исправлять» у существующих:

`sys_id` (см. выше), `sys_created_at`/`sys_created_by`, `sys_updated_at`/`sys_updated_by`, `application_id`, `sys_db_table_id`, `policy`.

## Именование (⚠️ КРИТИЧНО)

Нелокализуемые поля (машинные идентификаторы) — **только латиница, строчные буквы, snake_case**. Без кириллицы, заглавных, пробелов, camelCase.

| Поле | Пример |
|------|--------|
| `name` таблицы | `itsm_incident`, `c_crm_deal` |
| `column_name` | `assigned_to_id`, `is_active` |
| `value` (SysChoice) — database value | `active`, `in_progress` |
| `sys_name` (Choice) | `state_active` |
| `name` роли | `itil_admin` |
| `name` свойства (dot-notation) | `sdlc.ai_chat.nexus` |

> Choice `value` — это database value: `"in_progress"`, не `"In Progress"`/`"В работе"`. См. `data-structure`.

## Таблицы

- **Наследование** через `parent_id`: дочерняя таблица получает все колонки родителя. Создавая колонку на дочерней — проверь, нет ли одноимённой у родителя (иначе ошибка уникальности при деплое).
- **Префикс имени**: в продуктовом приложении — продуктовый (`pda_`/`itsm_`/`crm_`, опц. `c_`); `sys_*` — зона платформы и SA, в `sa/` с вендорскими правами такие таблицы создавать можно (детали: `data-structure`).
- **Уникальные поля** (`name` у таблиц/ролей/колонок, `sys_name` у choice): перед созданием записи проверь в `domain/`, нет ли записи с тем же значением — если есть, модифицируй её, а не создавай дубликат.

## Локализация

`SourceMessage` + `Message` — для **самостоятельных текстовых строк приложения**, **НЕ** для полей сущностей. Поля типа `Translated Text` (`title` таблицы/колонки/choice) переводятся через `sys_translation`. Детали: `localization`.

## Порядок зависимостей при деплое (⚠️ КРИТИЧНО)

Платформа импортирует записи последовательно. Если A ссылается на B (reference-поле), B создаётся **раньше** A — иначе ошибка `"<field>_id должно быть целым числом (Integer)"`. При такой ошибке — переставить тип-источник раньше зависящего согласно порядку ниже.

Порядок импорта типов (первые → последние); в скобках — от чего зависит:

`sys_language` · `sys_db_table` · `sys_filter_option_dynamic`(table) · `sys_db_column`(table, filter_option_dynamic) · `sys_db_column_override`(column) · `sys_choice`(table, column) · `sys_number`(table, column) · `sys_translation`(column, language, целевая запись) · `sys_state_model`(table) · `sys_state_transition`(state_model) · `sys_role` · `sys_user_criteria` · `sys_security_acl`(table, column, role) · `source_message` · `message`(source_message) · `sys_property` · `sys_script_include` · `sys_business_rule`(table) · `sys_script_client`(table, column) · `sys_ui_action`(table) · `sys_widget` · `sys_widget_instance`(widget) · `sys_ui_form`(table) · `sys_ui_form_section`(form) · `sys_ui_form_element`(form_section, column, widget_instance) · `sys_ui_related_list`(form, table) · `sys_ui_related_list_element`(related_list, column) · `sys_ui_list`(table) · `sys_ui_list_column`(ui_list, column) · `sys_notification_template`(table, language; RU-шаблон до EN с `main_template_id`) · `sys_notification_rule`(table, template) · `sys_notification_script` · `sys_menu_category` · `sys_menu_item`(category, table, role) · `sys_re_collection`(table опц.) · `sys_re_model`(table) · `sys_re_attribute`(re_model, re_collection через `re_container_id`) · `sys_re_choice`(re_attribute; наследует `column_id` от `SysChoice`) · `sys_re_model_client_script`(re_model/collection/attribute) · `sys_re_form_element`(re_model/collection/attribute) · `sys_rmc_catalog` · `sys_rmc_category`(catalog) · `sys_rmc_category_user_criteria`(category, user_criteria) · `sys_rmc_model`(category, table) · `sys_rmc_model_user_criteria`(model, user_criteria) · `wf_workflow`(table) · `wf_workflow_version`(workflow) · `wf_activity`*(workflow_version, activity_type) · `wf_activity_exit`(activity, activity_type_exit) · `wf_transition`(activity_exit, activity)

**Воркфлоу деплоится не порядком записей, а сценарием.** Опубликованную версию платформа не даёт править: SimpleGen делает `check out` (платформа копирует граф в новую версию с новыми `sys_id`), отправляет изменения в копию и публикует её. Поэтому `sys_id` активностей, выходов и переходов меняются после каждого деплоя — это нормально, их перезаписывает сам SimpleGen. Детали — `workflows`.

## Деплой

**НИКОГДА** не вноси изменения **конфигурации** на стенд напрямую (REST API, curl, интерфейс). Единственный путь: DSL-файлы → коммит → `simplegen.product_patch`. Прямые изменения рассинхронизируют DSL и стенд, ломают VCS-паки.

**Единственное исключение — runtime-демоданные.** Записи для демонстрации (`employee`, `sys_user_has_role`, записи бизнес-таблиц) конфигурацией не являются, в DSL и в SOP не попадают и создаются командой `/simplegen.demo_data_apply` по `_demo_data.json` (детали — `demo-rules`). На DSL-сущности (таблицы, колонки, роли, ACL, скрипты, формы, виджеты, меню) исключение **не распространяется**.

**Остальные записи стенда агент по своей инициативе тоже не меняет.** Запрос `POST`/`PUT`/`PATCH`/`DELETE` к Table API (curl, скрипт) — только по прямой просьбе пользователя в диалоге и после его подтверждения. Читать стенд можно. Перед любым запросом к стенду загрузить `table-api-rules`: там что считается прямой просьбой, авторизация и параметры запросов.

### Ошибки деплоя, которые не исправить правкой DSL

Если `simplegen patch`/`build`/`rollback` упал (в stderr `[error] …`, JSON нет) и в сообщении есть фрагмент из таблицы, дело в окружении или стенде, а не в коде. `code-author` не запускать и сборку не повторять: показать сообщение и действие, затем остановиться.

| Фрагмент в `[error] …` | Действие |
| ---------------------- | -------- |
| `requires an active task` | `/simplegen.context_set <task_id>` |
| `Repository not initialized`, `Package version changed since last init` | `/simplegen.init` |
| `is older than the version used to initialize this repo` | `npm update @simplegen/simplegen` |
| `Missing SimpleOne baseUrl`, `Missing SimpleOne credentials`, `Authentication failed` | Проверить в `.env` `SIMPLEONE_URL` и `SIMPLEONE_API_KEY` (или `SIMPLEONE_LOGIN` + `SIMPLEONE_PASSWORD`) |
| `SIMPLEONE_LOGIN is required`, `User with username` | Задать в `.env` `SIMPLEONE_LOGIN` — логин пользователя стенда. `patch` и `build` нужен и при `SIMPLEONE_API_KEY` |
| `SIMPLEONE_ELEVATED_ROLES lists` | Убрать из `SIMPLEONE_ELEVATED_ROLES` в `.env` роли, которых нет в `available: …`, или выдать их пользователю на стенде |
| `lack of rights`, `ACL restrictions`, `no access to delete`, `контроля доступа`, `Недостаточно прав` | Стенд отказал в записи по правам. Если сообщение называет роли (`can also elevate to …`), добавить нужную в `SIMPLEONE_ELEVATED_ROLES`. Если не называет, запросить права у администратора стенда |
| `Cannot read the state of local pack` | Стенд не отдал состояние сопки из `last_patch.local_pack_id`: проверить, что стенд доступен, и повторить |
| `Local pack … not found on the instance` | Сопки с этим ID на стенде нет: проверить ID |
| `Unable to resolve sys_application` | На стенде нет приложения продукта: проверить стенд и имя каталога продукта |
| `Failed to rollback previous build`, `VCS setup failed` | Стенд не дал откатить прежнюю сопку задачи или открыть новую. Причина — в конце сообщения: разобраться на стенде или с администратором |
| `fetch failed`, `operation was aborted` | Стенд недоступен: проверить `SIMPLEONE_URL` и сеть, повторить |

`HTTP 404` без этих фрагментов (например, `Property "<table>.<col>" does not exist!`) — это ошибка DSL, она идёт в обычный цикл исправлений.

Следующие предупреждения в `warnings[]` тоже не про DSL: `Failed to export SOP`, `Failed to import SOP into retrieved pack`, `Could not identify the local pack created by the import`, `Failed to apply VCS record policies`, `Failed to record the build`. Их только перечислить в отчёте, `code-author` по ним не запускать.

## Редактирование документов и входных данных

- **Точечные правки** — редактировать только затронутые разделы. Не переписывать файл целиком, не «улучшать стиль» и не перегенерировать ради одной правки. Незатронутые формулировки, нумерацию и структуру оставлять как есть.
- При добавлении/удалении элементов в нумерованных списках (UC, FR) — пересчитать нумерацию последующих элементов и обновить все перекрёстные ссылки.
- При правках спецификации **не модифицировать исходное описание задачи** и не добавлять в него сгенерированный контент — изменения вносить в целевые разделы (UC, FR, модель данных).
- Изменяя фильтры (`filter_conditions`, `condition`, `dynamic_reference_qualifier`) — указывать режим **ЗАМЕНИТЬ** или **ДОПОЛНИТЬ**. По умолчанию не заменять без явного указания; режим не указан — уточнить.

**Где разрешены ручные правки**: `_spec.md` — только **до** создания `_task_plan.md` (после — через `/simplegen.spec_check` → `/simplegen.spec_plan`). Код в `<product>/domain/` — да. `_task_plan.md` — структуру (фазы, трассировку FR→задачи) не менять вручную (пересобирать через `/simplegen.spec_plan`); отметки выполнения вести вручную можно.

## Конфигурация simplegen

```jsonc
// .env — активная фича (машинно-локально, вне гита, ставится через `simplegen context`)
// SIMPLEGEN_ACTIVE_FEATURE=<product>/<task_id>

// <product>/tasks/<task_id>/simplegen.json — контекст задачи (коммитится)
{ "initial_commit_sha": "abc123..." }
```
