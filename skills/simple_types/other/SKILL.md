---
description: Краткие описания прочих версионируемых типов, не покрытых отдельными правилами. Загружать при работе с other/**.
paths: '**/other/**/*'
user-invocable: false
---

# Типы в папке other

`other/` содержит версионируемые сущности без полноценных правил генерации — экспортируются при `init`/`sync`, когда таблица не попадает в известную категорию. Формат — TypeScript, путь `{app}/other/<naivePluralize(toKebabCase(tableTitle))>.ts`, константы в camelCase. `sys_id` — только в экспорте.

> Источник истины по полям — TypeScript-класс в `platform/data-structure/columns/<table>.ts`. Детальных правил нет: опираться на класс, существующие экспортированные записи и `project-core-rule`. Не выдумывать свойства; учитывать порядок деплоя по Reference-полям; при неуверенности — уточнять.

## Каталог типов (файл — класс — зависит от — описание)

**Workflow/процессы**: `workflow-activity-types` `WfActivityType` (типы активностей конструктора; поле `activity_table_id` задаёт дочернюю таблицу `wf_activity`, из которой берётся класс активности); `workflow-activity-type-exits` `WfActivityTypeExit` ← activity-types (выходы-ветвления, копируются в активность при её создании); `workflow-activity-type-categories` `WfActivityTypeCategory` (категории в палитре); `approval-rules` `SysApprovalRule` ← sys_db_table (правила согласования); `event-registers` `SysEventRegister` (именованные триггеры); `event-script-actions` `SysEventScriptAction` ← event-registers (скрипты на событие); `engines` `SysEngine` (процессные движки); `activity-types` `SysActivityType` (типы активностей процессов).

> Сами воркфлоу лежат не здесь, а в `workflows/` — правило `workflows`. Записи `wf_activity_type*` — платформенный каталог, на который ссылаются активности; создавать свои типы (кастомные активности) — отдельная задача, см. [Custom Activities](https://docs.simpleone.ru/platform/developer/business-logic/workflows/activities/custom).

**CMDB**: `ci-types` `SysCiType`; `ci-relationship-types` `SysCiRelationshipType` (типы связей КЕ); `cmdb-class` `CmdbClass` (иерархия классов CI).

**UI/отображение**: `column-style-rules` `SysColumnStyleRule` ← table,column (условное форматирование списков); `field-tips` `SysFieldTip` ← table,column (help-подсказки); `form-labels` `SysFormLabel` ← table; `inline-editing-configurations` `SysInlineEditingConfiguration` ← table; `ui-formatters` `SysUiFormatter` ← table (HTML/виджет вместо поля); `condition-builder-options` `SysConditionBuilderOption` (операторы конструктора условий); `carousel-images` `SysCarouselImage` (баннеры).

**Оформление**: `palettes` `SysPalette`; `palette-colors` `SysPaletteColor` ← palettes; `system-colors` `SysSystemColor` (глобальные цвета).

**Интеграция/импорт**: `import-sources` `SysImportSource` (JDBC/CSV/REST); `transform-maps` `SysTransformMap` ← import-sources,table; `transform-scripts` `SysTransformScript` ← transform-maps; `field-maps` `SysFieldMap` ← transform-maps; `ldap-servers` `SysLdapServer`; `ldap-settings` `SysLdapSetting` ← ldap-servers; `ldap-urls` `SysLdapUrl` ← ldap-servers.

**SLM/контракты**: `service-types` `SysServiceType`; `commitment-types` `SysCommitmentType` (SLA-обязательства); `contract-types` `SysContractType`.

**Портал**: `portal-cards` `PortalCard` (карточки сводок).

**Региональные**: `currencies` `SysCurrency`; `timezones` `SysTimezone`; `date-formats` `SysDateFormat`; `fiscal-periods` `SysFiscalPeriod`.

**Приложения/структура**: `applications` `SysApplication`; `trees` `SysTree` ← table; `levels` `SysLevel` (приоритет/влияние/срочность); `model-collections` `SysModelCollection`; `nodes` `SysNode` ← trees,node-types; `node-types` `SysNodeType`.

**Доступ/безопасность**: `role-contains` `SysRoleContains` ← sys_role (вложенность ролей); `policy-choices` `SysPolicyChoice`; `url-security-rules` `SysUrlSecurityRule` ← sys_role.

**Email**: `email-accounts` `SysEmailAccount` (SMTP/IMAP).

**Скрипты/расширения**: `server-scripts` `SysServerScript` (Script Actions); `scripted-related-lists` `SysScriptedRelatedList` ← table.

**Контент/БЗ**: `content-dbs` `SysContentDb`; `content-item-class` `SysContentItemClass` ← content-dbs.

**Поиск**: `text-search-groups` `SysTextSearchGroup`; `text-search-tables` `SysTextSearchTable` ← search-groups,table.

**Данные/маршрутизация**: `dynamic-default-values` `SysDynamicDefaultValue` ← table,column (скриптовые начальные значения); `dynamic-menu-categories` `SysDynamicMenuCategory`; `option-values` `SysOptionValue` ← options; `options` `SysOption`; `m2ms` `SysM2m` ← table (промежуточные M2M-таблицы); `routing-parameters` `SysRoutingParameter` ← table (автоназначение).
