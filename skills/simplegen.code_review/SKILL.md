---
description: Ревью сгенерированного кода — соответствие спецификации и плану, качество, производительность, SimpleOne API. Запускать после генерации, перед сборкой.
---

## Входные данные

```text
$ARGUMENTS
```

You **MUST** consider the user input before proceeding.
WRITE EVERYTHING IN RUSSIAN.

---

## Назначение

On-demand ревью кода активной задачи: один прогон `code-reviewer`, затем показ результата пользователю. `_spec.md` и `_task_plan.md` — единственные источники правды.

`<product>` — любой продукт, включая `sa/`.

## Шаг 1: Проверка версии

`version` из `.simplegen/state.json` vs `simplegen --version`: CLI ниже — `npm update @simplegen/simplegen`; выше — `/simplegen.init`; совпадают — продолжить.

## Шаг 2: Ревью

Прочитать активную фичу (`SIMPLEGEN_ACTIVE_FEATURE` в `.env`), затем `initial_commit_sha` из `<product>/<current_feature_path>/simplegen.json`. Нет активной задачи — сообщить. Нет `initial_commit_sha` — предупредить, использовать текущий HEAD. Запустить субагента `code-reviewer`.

- `VERDICT: PASS` → вывести «✅ Code Review пройден», предложить `/simplegen.product_patch`.
- Иначе → показать отчёт и scorecard пользователю.

PASS не отменяет ревью человеком: указать это в выводе. Проверяется DSL в ветке — SOP на стенд для ревью не загружается.

## Следующий шаг

→ `/simplegen.code_generate` — исправить замечания
→ `/simplegen.product_patch` — собрать и задеплоить
