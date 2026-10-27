---
description: Стандарты email-уведомлений — правила, шаблоны писем, скрипты условий и получателей. Загружать при работе с business-logic/notifications/**.
paths: '**/notifications/**/*'
user-invocable: false
---

# Email-уведомления

DSL-экспорт: TypeScript + HTML-шаблоны писем. Классы: `SysNotificationRule`, `SysNotificationTemplate`, `SysNotificationScript`. `sys_id` — только в экспорте. [Docs](https://docs.simpleone.ru/platform/admin-help/notifications/), скрипты — `script-rules` §9.

Notification Script — серверный контекст, спец-глобалы `current` (источник), `template` (`SimpleTemplatePrinter`), `email` (`SimpleEmailOutbound`), `event` (`sys_event`).

## Порядок импорта (⚠️ КРИТИЧНО)

`sys_notification_script` → `sys_notification_template` (Main/RU) → `sys_notification_template` (Translation/EN, через `main_template_id`) → `sys_notification_rule`. Scripts до Rules (ссылка `${script:...}`); Main-шаблон (RU) до перевода (EN).

Уникальность: rule `name`+`table_id`; template `name`+`language_id` и `main_template_id`+`language_id`; script `name` (глобально, используется в `${script:name}`).

## Размещение и поля

- **Scripts** — `business-logic/notifications/scripts/records.ts`. Поля: `name`✱ (для `${script:name}`), `script` (inline IIFE с `current, template, email, event`, возвращает строку email через запятую), `policy`.
- **Templates** — `business-logic/notifications/{table.name}/templates.ts` (+ `.html` рядом). Поля: `table_id`✱, `name`, `language_id`✱ (импорт `SysLanguage` из `localization/languages.ts`, **не** hardcoded), `main_template_id` (null для Main/RU, ссылка на Main для перевода), `subject` (поддерживает `${current.field}`), `message_html` (импорт `.html` с `as unknown as string`), `message_text`, `policy`.
- **Rules** — `business-logic/notifications/{table.name}/rules.ts`. Поля: `table_id`✱, `name`, `send_when` (`"record"`), `inserted`/`updated`, `condition` (Encoded Query, см. `business-logic`), `notification_template_id` (ссылка на **Main** шаблон), `other` (получатели), `send_self`, `active`, `message_html`, `policy`.

Константа — `toCamelCase(name [+ language для шаблонов], 50)`.

**`other`** (получатели): `${current.field.email}`, `${current.field.responsible.email}`, `${script:scriptName}`, несколько через запятую.

## Примеры

```typescript
// templates.ts (RU + EN)
import { languageRu, languageEn } from "../../../localization/languages";
import ruHtml from "./story-completed-ru.html";
import enHtml from "./story-completed-en.html";
export const demoStoryCompletedRu: SysNotificationTemplate = new SysNotificationTemplate({
    "table_id": demoStory, "name": "Demo Story Completed", "language_id": languageRu, "main_template_id": null,
    "subject": "История завершена: ${current.number}", "message_html": ruHtml as unknown as string, "message_text": "", "policy": "Open",
});
export const demoStoryCompletedEn: SysNotificationTemplate = new SysNotificationTemplate({
    "table_id": demoStory, "name": "Demo Story Completed", "language_id": languageEn, "main_template_id": demoStoryCompletedRu,
    "subject": "Story Completed: ${current.number}", "message_html": enHtml as unknown as string, "message_text": "", "policy": "Open",
});

// rules.ts
export const demoStoryCompleted: SysNotificationRule = new SysNotificationRule({
    "send_when": "record", "table_id": demoStory, "name": "Demo Story Completed", "active": true, "send_self": true,
    "inserted": false, "updated": true, "condition": "(demo_stageCHANGESTOdone)",
    "other": "${current.demo_assignee.email}, ${current.demo_group.responsible.email}",
    "notification_template_id": demoStoryCompletedRu, "message_html": ruHtml as unknown as string, "policy": "Open",
});

// scripts/records.ts — inline IIFE
export const getDemoAdminEmails: SysNotificationScript = new SysNotificationScript({
    "name": "getDemoAdminEmails", "policy": "Open",
    "script": "(function runNotificationScript(current, template, email, event) { /* собрать emails */ return [...emails].join(','); })(current, template, email, event);",
});
```
```html
<!-- story-completed-ru.html — ${current.field} для значений, .getDisplayValue() для reference -->
<p>История <b>${current.number}</b> завершена. Тема: ${current.subject}</p>
```
