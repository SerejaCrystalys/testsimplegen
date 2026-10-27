---
description: Диспетчер правил — индекс «путь в domain/ или тип сущности → нужный rules-файл». Читать ПЕРВЫМ при любой генерации или правке DSL, чтобы грузить только релевантные simple_types, а не все ~1200 строк сразу.
user-invocable: false
---
# Диспетчер правил (rule-router)

Точка входа в **выборочную** загрузку правил. Цель — не тянуть в контекст все `simple_types/` (это ~1200 строк), а грузить только то, что нужно для конкретной задачи.

## Где лежат правила

Сквозные правила — в `.claude/skills/<имя>/SKILL.md`, правила типов сущностей — в `.claude/skills/simple_types/<имя>/SKILL.md`. В таблицах ниже правила указаны **по имени**; грузи файл по имени, взяв путь из этих двух вариантов.

## Протокол загрузки

1. **Всегда**: `project-core-rule` (поведение, sys_id, именование, порядок деплоя) — уже always-on, отдельно грузить не нужно.
2. Определи **затронутые пути** в `<product>/domain/` (или разделы `_spec.md`) для текущей задачи.
3. Для каждого пути/раздела загрузи **только** сматченное правило из таблицы. Нерелевантные `simple_types` НЕ грузить.
4. Если задача включает **скрипт** (`.js`: BR, Client Script, ACL, UI Action, Script Include, Scheduled Script, Notification Script, Widget) — добавь `script-rules` (канон API-справочника и запрещённых паттернов).
5. Если задача про **виджет** или в постановке есть **Figma-ссылка** — добавь `widgets` + `design-system-rules`.
6. Если генерируешь **интерфейсные тексты** (заголовки, подсказки, сообщения; правка `_spec.md`/`_task_plan.md`) — добавь `interface-text-rules`.

## Таблица маршрутизации (сущность / путь → правило)

| Путь в `domain/` | Раздел спеки / сущность | Правило |
|------------------|-------------------------|---------|
| `data-structure/**` (tables, columns, choice-options, state-models, state-transitions, record-numbers, column-overrides) | Таблицы, Колонки, Choices, State Model, нумерация | `data-structure` |
| `data-structure/dynamic-filter-options/**` | Динамический фильтр | `dynamic-filter` |
| `business-logic/business-rules/**`, `business-logic/client-scripts/**` | Бизнес-правила, Client Scripts | `business-logic` (+ `script-rules`) |
| `business-logic/widgets/**` | Виджет | `widgets` + `design-system-rules` (+ `script-rules`) |
| `business-logic/script-includes/**` | Script Include | `script-includes` (+ `script-rules`) |
| `business-logic/notifications/**` | Уведомления, шаблоны писем | `notifications` (+ `script-rules` для скриптов) |
| `business-logic/system-properties.ts`, `business-logic/property-categories.ts` | System Property | `system-properties` |
| `business-logic/ui-actions/**` | UI Action / кнопка | `ui-actions` (+ `script-rules`) |
| `access-control/roles.ts`, `access-control/security-acls/**` | Роли, ACL, матрица доступов | `access-control` (+ `script-rules` для ACL-скриптов) |
| `access-control/user-criterias.ts` | User Criteria | `user-criteria` |
| `forms/**` | Форма, секции, поля, Related Lists | `forms` |
| `workflows/**` | Воркфлоу, активности, выходы, переходы, согласования | `workflows` (+ `script-rules` для скриптов активностей) |
| `lists/**` | List View, колонки списка | `lists` |
| `rem/**` | REM — модель/коллекция/атрибут/choice | `record-extension-model` (+ `script-rules` для скриптов модели) |
| `schedule/**` | Расписания, Scheduled Jobs/Imports | `schedules` (+ `script-rules`) |
| `navigation/**` | Меню (категории, пункты) | `menus` |
| `service-catalog/**` | Каталог услуг (Catalog/Category/Model) | `service-catalog` |
| `portal/**` | Портал (Pages/Containers/Portals/Nodes) | `portal` (+ `widgets` для инстансов виджетов) |
| `localization/**` | Языки, Source Message, переводы | `localization` (+ `interface-text-rules`) |
| `other/**` | Прочие версионируемые сущности | `other` |

## Сквозные правила (по триггеру, не по пути)

| Триггер | Правило |
|---------|---------|
| Любой `.js`-скрипт | `script-rules` — API-справочник, запрещённые паттерны, контекст выполнения |
| Виджет или Figma-ссылка в постановке | `design-system-rules` + `widgets` |
| Генерация/правка интерфейсных текстов (RU/EN) | `interface-text-rules` |
| Генерация диаграмм (`/simplegen.diagrams_build`, `*.puml`, `*_diagrams.md`) | `diagram-rules` — тип под цель, тема, именование файлов |
| Требование «поле не редактируется / заполняется системой» | `data-structure` (выбор механизма) + `access-control` (write-ACL) + `business-logic` (клиентский скрипт) |
| Любая команда `/simplegen.demo_*`, правка `_demo_plan.md` / `_demo_data.json` / `_demo_log.md`, тесты на стенде (`test-author`, `tasks/*/_local/**`, `tests/e2e/**`) | `demo-rules` — работа с браузером, авторизация, имперсонация и возврат под себя |
| `/simplegen.ultra`, флаг `--ultra`, строка `Режим: ultra:*` в задании, `_ultra_log.md` | `ultra-rules` — подрежимы, блоки ответа, допущения, стоп-условия, журнал |

> Правило: грузи **минимально достаточный** набор. Сомневаешься, нужен ли файл, — открой его описание (frontmatter `description`), а не весь текст.
