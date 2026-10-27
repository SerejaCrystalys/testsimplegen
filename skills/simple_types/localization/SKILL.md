---
description: Стандарты локализации — языки, Source Message, Message, Translation. Загружать при работе с localization/** (плюс interface-text-rules).
paths: '**/localization/**/*'
user-invocable: false
---

# Локализации

DSL-экспорт, TypeScript. Классы: `SourceMessage`, `Message`, `SysLanguage`, `SysTranslation`. `sys_id` — только в экспорте. [Docs](https://docs.simpleone.ru/platform/admin-help/localization-and-multi-language-support#localize-messages). Как указывать в спецификации — [spec_create.md](../../commands/simplegen.spec_create.md#1-локализация-новых-полей).

Размещение: `localization/languages.ts` (`SysLanguage`); `localization/source-messages/<slug>/source.ts` + `messages.ts` (slug = kebab-case от `message`, при пустом — sys_id); `localization/translations/<table>/<column>.ts` (`SysTranslation`).

Порядок деплоя: `SysLanguage` → `SysTranslation` (после `sys_db_column`, `sys_language`) → `SourceMessage` → `Message` (после `SourceMessage`).

Уникальность: `source_message` — `message`+`category`+`application_id`; `message` — `source_message_id`+`language`; `sys_language` — `language` (ISO-код, глобально); `sys_translation` — `record_id`+`column_id`+`language_id`.

## SourceMessage vs Translated Text (⚠️ КРИТИЧНО)

`SourceMessage` + `Message` — **только** для самостоятельных текстовых строк приложения (категория `"app"`: тексты UI, кастомные сообщения, строки виджетов). **НИКОГДА** не создавать для полей сущностей.

Поля **Translated Text** (`SysDbTable.title`, `SysDbColumn.title`, `SysChoice.title`, `SysMenuCategory.title`, `SysMenuItem.title`, `SysUiFormSection.title` и т.п.) переводятся платформой через `sys_translation`. Обычный `String` (`SysBusinessRule.name`) перевода не требует.

## SysTranslation

Переводы значений полей Translated Text для конкретных записей. Группировка по таблице и колонке. Константа — `toCamelCase(...)`.

Поля: `column_id`✱ (переводимое поле — определяет путь), `record_id`✱ (**полиморфная** ссылка на запись в таблице, которой принадлежит колонка), `language_id`✱ (`SysLanguage`), `value` (текст перевода), `policy`.

⚠️ **Путь определяется по таблице, которой принадлежит колонка, а не по переводимой записи**. Перевод названия таблицы `pda_project` → `translations/sys_db_table/title.ts` (т.к. `title` принадлежит `sys_db_table`). Аналогично: заголовок колонки → `translations/sys_db_column/title.ts`; метка choice → `translations/sys_choice/title.ts`.

**Изменить vs переопределить**: существующий перевод в `<product>/domain/` — изменить `value` (не дублировать); перевод из `platform/`/`sa/` — новая запись с `overrides_by`.

## Примеры

```typescript
// languages.ts
export const languageRu: SysLanguage = new SysLanguage({ "name": "Русский", "language": "ru", "number_format": "default", "policy": "Open" });
export const languageEn: SysLanguage = new SysLanguage({ "name": "English", "language": "en", "number_format": "default", "policy": "Open" });

// translations/sys_db_table/title.ts — переводы названий таблиц
import { SysTranslation } from "../../../../../platform/data-structure/columns/sys_translation";
import { title } from "../../../../../platform/data-structure/columns/sys_db_table";
import { pdaProject } from "../../../data-structure/tables/pda_project";
import { languageRu } from "../../languages";
export const pdaProjectLabelRu: SysTranslation = new SysTranslation({
    "column_id": title, "record_id": pdaProject, "language_id": languageRu, "value": "Проект", "policy": "Open",
});

// source-messages/complete-the-project/source.ts + messages.ts
export const source_message_x: SourceMessage = new SourceMessage({ "message": "Complete the project", "category": "app", "policy": "Protected" });
export const message_ru: Message = new Message({ "source_message_id": source_message_x, "language": "ru", "translation": "Завершение проекта", "policy": "Open" });
```

Практики: не менять `message` после создания (изменит slug/путь); искать существующее сообщение перед созданием (дубликат → конфликт); импортировать `SysLanguage` из `languages.ts`, не hardcoded sys_id; экспортировать все активные языки.
