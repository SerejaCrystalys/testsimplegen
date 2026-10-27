---
description: Стандарты календарей и запланированных заданий — Calendars, Schedule Elements, Scheduled Jobs и Imports, скрипты. Загружать при работе с schedule/** (плюс script-rules).
paths: '**/schedule/**/*'
user-invocable: false
---

# Календари и запланированные задания

DSL-экспорт: TypeScript + JS-скрипты. Классы: `SysSchedule`, `SysScheduleElement`, `SysScheduleJob`, `SysScheduleImport`, `SysScheduleScript`. `sys_id` — только в экспорте. [Docs](https://docs.simpleone.ru/ru/platform/admin-help/date-and-time-system/schedules), [SimpleSchedule API](https://docs.simpleone.ru/ru/platform/developer-help/developer-api/server-side-api/simpleschedule).

Две независимые области: **Календари** (`sys_schedule` + `sys_schedule_element`) — рабочее время, SLA; **Задания** (`sys_schedule_job` ← `sys_schedule_import`/`sys_schedule_script`) — запуск по расписанию.

Порядок импорта: `sys_schedule` (→ `sys_timezone` если указан) → `sys_schedule_element`. Задания: → `sys_user` (`run_as_user_id`), импорты → источник. Задания не зависят от календарей. Константа — `toCamelCase(name, 50)`.

## Форматы значений (⚠️ КРИТИЧНО)

DateTime `YYYY-MM-DD HH:mm:ss`; Date `YYYY-MM-DD`; Time `HH:mm:ss`; Duration `D HH:mm:ss` (`"0 01:30:00"` = 1.5ч). `days_of_week` — строка через запятую без пробелов: `monday,tuesday,...,sunday` (`run_day_of_week` — одно из тех же значений; `run_day_of_month` — `"1"`..`"31"`).

Пустые значения: `parent_id`/`timezone_id`/`days_of_week`/`repeat_until_on`/`repeat_count`/`run_*` → `null` (не `""`/`0`); `description` → `""` (не null). Неиспользуемые поля расписания (по `run_type`) — `null`.

## Календари

- **`SysSchedule`** (`schedule/schedules.ts`): `name`✱, `description`, `parent_id` (родитель — наследует элементы), `timezone_id` (null = UTC), `included_schedule_ids` (List, глубина 2; null если нет), `policy`. Иерархия: без циклов, не включать себя, parent ≠ included.
- **`SysScheduleElement`** (`schedule/schedule-elements.ts`): `name`, `schedule_id`, `type`, `started_at`/`finished_at` (DateTime первого интервала), `repeat_type`, `days_of_week`, `repeat_until_on` (Date; null = бесконечно; наступление прерывает интервал), `repeat_count`, `policy`.

`type` (database values): `""` (рабочее), `"meeting"`/`"call"`/`"conference"` (рабочее, личный календарь), `"exclude"` (нерабочее: праздники), `"time_off"` (отгул). `repeat_type`: `"none"`/`"daily"`/`"weekly"`/`"monthly"`/`"yearly"`.

⚠️ **Ночные смены**: интервал через полночь разбивать на **два** элемента (до и после `00:00:00`); `repeat_until_on` второго — на день позже.

### SimpleSchedule API (серверный контекст)

```javascript
const schedule = new SimpleSchedule('157165292607666710', 'UTC');  // или new SimpleSchedule(); schedule.load(id, tz)
schedule.isWorkingTime(dt);   // true только для рабочих сегментов
schedule.isInSchedule(dt);    // true для ЛЮБЫХ сегментов, включая exclude/time_off
schedule.duration(start, end);          // рабочее время между датами (даты должны быть в расписании — проверять isInSchedule)
schedule.whenWillExpire(start, secs);   // когда истекут N рабочих секунд (не поддерживает отрицательные; null если !isValid())
schedule.whenNext(dt, '');              // время до следующего рабочего интервала
```
⚠️ Для «сейчас рабочее время?» — `isWorkingTime()`, не `isInSchedule()`.

## Задания (наследуются от `sys_schedule_job` — напрямую в него не писать)

- **`SysScheduleScript`** (`schedule/scheduled-scripts/records.ts` + `.js`): `name`✱, `run_as_user_id`✱ (нужны права для CRUD), `active`, `job` (описание), `script`, `run_type`✱, + поля по типу.
- **`SysScheduleImport`** (`schedule/scheduled-imports.ts`): `name`✱, `import_source_id`✱, `run_as_user_id`✱, `run_type`✱ + поля по типу.

**`run_type` → обязательные поля**: `daily` → `run_time`; `weekly` → `run_time`+`run_day_of_week`; `monthly` → `run_time`+`run_day_of_month`; `once` → `run_start_at`; `periodically` → `run_start_at`+`run_period`; `on_demand` → ничего.

Ограничения выполнения: ≤20 параллельных (10 импортов + 10 скриптов); лимит 12ч; проверка раз в минуту.

Translated Text (через `sys_translation`, не SourceMessage): `name`/`description` у `sys_schedule`/`sys_schedule_element`, `name`/`job` у заданий.

## Примеры

```typescript
// schedules.ts
export const standardWorkWeek: SysSchedule = new SysSchedule({
    "name": "Standard Work Week 8x5", "description": "5×8ч без праздников", "parent_id": null,
    "timezone_id": null, "included_schedule_ids": null, "policy": "Open",
});
// schedule-elements.ts
export const workingMorning: SysScheduleElement = new SysScheduleElement({
    "name": "Рабочее утро", "schedule_id": standardWorkWeek, "type": "",
    "started_at": "2026-01-05 09:00:00", "finished_at": "2026-01-05 13:00:00",
    "repeat_type": "weekly", "days_of_week": "monday,tuesday,wednesday,thursday,friday", "repeat_until_on": null, "policy": "Open",
});
export const newYearHolidays: SysScheduleElement = new SysScheduleElement({
    "name": "Новогодние каникулы", "schedule_id": standardWorkWeek, "type": "exclude",
    "started_at": "2026-01-01 00:00:00", "finished_at": "2026-01-09 00:00:00", "repeat_type": "yearly", "policy": "Open",
});
// scheduled-scripts/records.ts
import { systemAdmin } from "../../../../platform/data-structure/users/sys_user";
import dailyCleanup from "./daily-cleanup-old-records.js";
export const dailyCleanupOldRecords: SysScheduleScript = new SysScheduleScript({
    "name": "Daily cleanup old records", "run_as_user_id": systemAdmin, "active": true, "job": "Удаление просроченных",
    "run_type": "daily", "run_time": "02:00:00", "run_start_at": null, "run_day_of_week": null, "run_day_of_month": null,
    "run_period": null, "script": dailyCleanup as unknown as string, "policy": "Open",
});
// scheduled-imports.ts — import_source_id обязателен
export const dailyUserSync: SysScheduleImport = new SysScheduleImport({
    "name": "Daily User Sync", "import_source_id": ldapUserSource, "run_as_user_id": systemAdmin, "active": true,
    "run_type": "daily", "run_time": "03:00:00", "run_period": null, "policy": "Open",
});
```
```javascript
// daily-cleanup-old-records.js (имя файла — toKebabCase(name, 50))
export default function script() {
  (function executeScheduledScript() {
    const cutoff = new SimpleDateTime(); cutoff.addDays(-90);
    const r = new SimpleRecord('demo_archive');
    r.addQuery('sys_created_at', '<', cutoff.getValue()); r.addQuery('state', 'closed'); r.query();
    while (r.next()) r.deleteRecord();
  })();
}
```

Практики: наследование (`parent_id`/`included_schedule_ids`) вместо дублирования; `repeat_until_on` с запасом или null; явный `timezone_id` для локаций; `on_demand` для отладки; сложную логику скриптов — в Script Includes.
