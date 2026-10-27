---
description: Переключает активную фичу/задачу (SIMPLEGEN_ACTIVE_FEATURE в .env). Запускать перед командами, которым нужна активная задача (code_generate, product_patch, demo_*).
---

## Входные данные

```text
$ARGUMENTS
```

You **MUST** consider the user input before proceeding.
WRITE EVERYTHING IN RUSSIAN.

---

## Назначение

Команда `/simplegen.context_set` устанавливает текущую активную фичу, с которой будут работать остальные команды simplegen.

> **Примечание**: `<product>` в этом документе означает любой продукт, включая `sa/` с вендорскими правами. `<task_id>` — идентификатор задачи (без префикса `tasks/`), например `STR001234`.

---

## Синтаксис

```
/simplegen.context_set <task_id>
```

**Пример**:

```
/simplegen.context_set STR001234
```

---

## Алгоритм работы

### Шаг 0: Проверить версию simplegen (⚠️ ОБЯЗАТЕЛЬНО)

Сравнить `version` (`.simplegen/state.json`) с выводом `simplegen --version`: CLI ниже — `npm update @simplegen/simplegen`; CLI выше — `/simplegen.init`; совпадают — продолжить. Правило (канон) — `project-core-rule` (always-on).

### Шаг 1: Валидация пути

1. Проверить, что каталог `<product>/tasks/<task_id>/` существует
2. Проверить, что существует `<product>/tasks/<task_id>/simplegen.json`
   - Если нет — ошибка (файл создаёт `simplegen.spec_create`)

### Шаг 2: Установить активную фичу

Выполнить:

```
simplegen context <product> <task_id>
```

Команда запишет `SIMPLEGEN_ACTIVE_FEATURE=<product>/<task_id>` в `.env` (остальные переменные, включая `SIMPLEONE_*`, сохраняются). `.env` в `.gitignore` — контекст не попадает в гит.

### Шаг 3: Вывести сводку

Показать информацию о выбранной фиче:

- `initial_commit_sha` (из `<product>/tasks/<task_id>/simplegen.json`);
- Наличие текущего SOP (`[<PRODUCT>]_<task_id>.sop`);
- Наличие `_spec.md`, `_task_plan.md`, `_demo_plan.md`, `_demo_data.json` и `_demo_log.md`.

---

## Формат вывода

### Успех

```markdown
# ✅ Контекст установлен

**Активная фича**: `<task_id>`

## Информация о фиче

| Параметр           | Значение               |
| ------------------ | ---------------------- |
| initial_commit_sha | abc123...              |
| SOP                | `[SA]_STR001234.sop` ✅ |

## Файлы

| Файл              | Статус |
| ----------------- | ------ |
| `_spec.md`        | ✅ Есть |
| `_task_plan.md`   | ✅ Есть |
| `_demo_plan.md`   | ❌ Нет  |
| `_demo_data.json` | —      |
| `_demo_log.md`    | —      |

## Доступные команды

→ `/simplegen.spec_check` — проверить спецификацию
→ `/simplegen.spec_plan` — построить план
→ `/simplegen.code_generate` — генерировать код
→ `/simplegen.code_review` — review изменений
→ `/simplegen.demo_plan` — создать демо-сценарий
→ `/simplegen.demo_data_prepare` — подготовить демоданные
→ `/simplegen.demo_run` — запустить демонстрацию
```

### Каталог не найден

```markdown
⚠️ Каталог `<product>/tasks/<task_id>/` не найден.

Проверьте идентификатор или создайте новую фичу:
→ `/simplegen.spec_create [описание задачи]`
```

### simplegen.json не найден

```markdown
⚠️ Файл `<product>/tasks/<task_id>/simplegen.json` не найден.

Возможно, фича была создана вручную. Используйте:
→ `/simplegen.spec_create [описание]` — создать фичу правильно
```

---

## Примеры

### Пример 1: Установка контекста

**Input**:

```
/simplegen.context_set STR001234
```

**Output**:

```markdown
# ✅ Контекст установлен

**Активная фича**: `STR001234`

## Информация о фиче

| Параметр           | Значение             |
| ------------------ | -------------------- |
| initial_commit_sha | 7a3b2c1...           |
| SOP                | — (ещё не собирался) |

## Файлы

| Файл              | Статус |
| ----------------- | ------ |
| `_spec.md`        | ✅ Есть |
| `_task_plan.md`   | ❌ Нет  |
| `_demo_plan.md`   | ❌ Нет  |
| `_demo_data.json` | —      |
| `_demo_log.md`    | —      |

## Доступные команды

→ `/simplegen.spec_check` — проверить спецификацию
→ `/simplegen.spec_plan` — построить план
```

---

## Список задач

Если идентификатор не указан, показать список существующих задач:

```markdown
## Существующие задачи

| #   | Задача    | Статус                                       |
| --- | --------- | -------------------------------------------- |
| 1   | STR001234 | _spec.md ✅, _task_plan.md ❌, _demo_plan.md ❌ |
| 2   | STR005678 | _spec.md ✅, _task_plan.md ✅, _demo_plan.md ❌ |

Укажите идентификатор для установки контекста:
```

/simplegen.context_set <task_id>

```
```

---
