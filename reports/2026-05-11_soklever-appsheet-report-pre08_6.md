# Отчёт по брифингу pre-#08.6

**Дата выполнения:** 2026-05-11
**Версия проекта на входе:** AppSheet 1.000523
**Версия проекта на выходе:** AppSheet 1.000535 (обновление платформы в ходе сессии)
**Исполнитель:** Sonnet (без Adaptive thinking)

---

## §1. Стартовый чек-лист

| # | Вопрос | Ответ |
|---|--------|-------|
| 2.1 | Сегодняшняя физическая дата | 2026-05-11 |
| 2.2 | Текущая версия AppSheet | 1.000523 |
| 2.3 | Закрыт ли pre-#08.5? | ✅ Да |
| 2.4 | Изменения после pre-#08.5 | Нет |

---

## §2. Бэкап

| # | Действие | Имя | Статус |
|---|----------|-----|--------|
| 3.1.1 | AppSheet (до) | `slk-bkp-2026-05-11-pre-08-6` | ✅ Выполнен владельцем |
| 3.1.2 | Sheets (до) | `2026-05-11 soklever-google-ancestor-v6 (pre-08-6)` | ✅ Выполнен владельцем |
| 3.2.1 | AppSheet (после) | `slk-bkp-2026-05-11-post-08-6` | ⚠️ По решению владельца не создавался |
| 3.2.2 | Sheets (после) | `2026-05-11 soklever-google-ancestor-v6 (post-08-6)` | ⚠️ По решению владельца не создавался |

---

## §3. Задача §4.1 — Удаление `billing_frequency`

### Выполнено

| # | Шаг | Результат |
|---|-----|-----------|
| 4.1.1.1 | Sheets → лист `tariffs` → подтверждён столбец D = `billing_frequency` (Enum «Месяц»/«Разовый») | ✅ |
| 4.1.1.2 | Удалён столбец D полностью | ✅ |
| 4.1.1.3 | AppSheet Editor → Data → tariffs → Regenerate Schema | ✅ |
| 4.1.1.4 | Errors & Warnings после регенерации | ✅ 0 ошибок, 0 предупреждений |
| 4.1.1.5 | Глобальный поиск `billing_frequency` в Editor | ✅ 0 результатов (Views: нет, Data: нет) |
| 4.1.1.6 | Save | ✅ |

### Контрольная проверка

- Карточка тарифа открывается ✅
- Список тарифов отображается корректно ✅
- Связанные ученики в карточке тарифа видны ✅

---

## §4. Задача §4.2 — Valid_if на `charges[period]`

### Итоговые формулы

**Initial value:**
```
INDEX(settings[current_period], 1)
```

**Valid_if:**
```
SELECT(
  ref_periods[period],
  AND(
    NUMBER([period]) >= NUMBER(INDEX(settings[current_period], 1)),
    NUMBER([period]) <= NUMBER(INDEX(settings[current_period], 1)) + 3
  )
)
```

### Результат Test condition в Editor

Expression Result для каждой строки = `202606, 202607, 202608, 202609` — текущий открытый период + 3 следующих. Закрытые периоды (`202603`, `202604`, `202605`) в списке не появляются. ✅

### Дополнительно: `ref_periods` → Read-Only

В ходе выполнения §4.2 обнаружено: в dropdown присутствовала кнопка «+ New» (создание новой записи в `ref_periods`). По указанию стратега:

- `ref_periods` → Table settings → **Update mode = Read-Only** (сняты Updates, Adds, Deletes).
- `ref_periods` — справочник на 60 строк, наполнен заранее (#6, ТД-1). Добавлений и изменений руками не предполагается.
- После установки Read-Only кнопка «+ New» в dropdown исчезла. ✅

### Поведение dropdown

| # | Сценарий | Результат |
|---|----------|-----------|
| 4.2.4.1 | Форма «+ начисление» → поле `period` | `202606` подставлен автоматически ✅ |
| 4.2.4.2 | Dropdown `period` | Показывает `202606`, `202607`, `202608`, `202609`. Закрытые периоды отсутствуют ✅ |
| 4.2.4.3 | Кнопка «+ New» в dropdown | Отсутствует (ref_periods = Read-Only) ✅ |
| 4.2.4.4 | Save charge с текущим периодом | Успех ✅ |

---

## §5. Внебрифинговые правки (по указанию стратега/владельца)

### §5.1. Защита Edit + Delete от закрытого периода — `charges`, `payments`, `expenses`

**Контекст:** в ходе регрессии обнаружено:
1. Edit/Delete в карточке charge через «фильтр начислений» отсутствовали — slice `charges_filtered` имел Update mode = Read-Only.
2. Delete на `charges` не имел Show_If — был виден для закрытых периодов.
3. Аналогичные проблемы обнаружены на `payments` и `expenses`.

По указанию стратега выполнено распространение защиты «закрытый период = read-only» на Edit + Delete для трёх таблиц.

---

#### `charges`

**Системный Delete → Only if this condition is true:**
```
[period] = INDEX(settings[current_period], 1)
```

**Slice `charges_filtered` → Update mode:** включены Updates + Deletes (было Read-Only).

**Живой Detail view для маршрута «фильтр начислений»:** `charges_filtered_Detail` (system generated).

**Результаты тестов:**

| Маршрут | Период | Edit | Delete |
|---------|--------|------|--------|
| фильтр начислений | 202606 (текущий) | ✅ виден | ✅ виден |
| фильтр начислений | 202605 (закрытый) | ✅ скрыт | ✅ скрыт |

---

#### `payments`

**Системный Edit → Only if this condition is true:**
```
TEXT([payment_date], "YYYYMM") = INDEX(settings[current_period], 1)
```

**Системный Delete → Only if this condition is true:**
```
TEXT([payment_date], "YYYYMM") = INDEX(settings[current_period], 1)
```

**Системный Delete → Position:** изменён с Hide на **Prominent** (Delete для платежей должен быть доступен в открытом периоде — по решению владельца).

**Slice `payments_filtered` → Update mode:** включены Updates + Deletes (было Read-Only).

**Живой Detail view для маршрута «фильтр платежей»:** `payments_filtered_Detail` (system generated).

**Результаты тестов:**

| Маршрут | Период | Edit | Delete |
|---------|--------|------|--------|
| фильтр платежей | 6/2/2026 (июнь, текущий) | ✅ виден | ✅ виден |
| фильтр платежей | 5/2/2026 (май, закрытый) | ✅ скрыт | ✅ скрыт |

---

#### `expenses`

**Valid_if на `expenses[period]`** — исправлена формула (была `- 3` на нижней границе, убрано):
```
SELECT(
  ref_periods[period],
  AND(
    NUMBER([period]) >= NUMBER(INDEX(settings[current_period], 1)),
    NUMBER([period]) <= NUMBER(INDEX(settings[current_period], 1)) + 3
  )
)
```

**Системный Edit → Only if this condition is true:**
```
[period] = INDEX(settings[current_period], 1)
```

**Системный Delete → Only if this condition is true:**
```
[period] = INDEX(settings[current_period], 1)
```

**Slice `expenses_filtered` → Update mode:** включены Updates + Deletes (было Read-Only).

**Живой Detail view для маршрута «фильтр расходов»:** `expenses_filtered_Detail` (system generated).

**Результаты тестов:**

| Маршрут | Период | Edit | Delete |
|---------|--------|------|--------|
| фильтр расходов | 202606 (текущий) | ✅ виден | ✅ виден |
| фильтр расходов | 202603 (закрытый) | ✅ скрыт | ✅ скрыт |

---

### §5.2. Паттерн: slice Read-Only блокирует системные actions

**Обнаружено в ходе сессии:** три slice-а (`charges_filtered`, `payments_filtered`, `expenses_filtered`) имели Update mode = Read-Only. Это блокировало отображение системных Edit и Delete в slice-Detail view, даже при корректно настроенных Show_If/Only if на самих actions.

**Решение:** включить Updates + Deletes на уровне slice. Условия Show_If/Only if на actions при этом продолжают работать корректно — защита закрытого периода сохраняется.

**Для фиксации в SKILL:** добавить в appsheet-setup-SKILL.md паттерн: если системный action не отображается в slice-Detail — проверить Update mode на slice.

---

## §6. Регрессия §6.3

| # | Шаг | Результат |
|---|-----|-----------|
| 6.3.1 | Дашборд — 4 плитки | ✅ Работают |
| 6.3.2 | Кнопка «Закрыть период» | ✅ Видна, не нажималась |
| 6.3.3 | Карточки учеников, тарифов, семей | ✅ Открываются |
| 6.3.4 | Edit charge из текущего периода (оба маршрута) | ✅ Виден, Save работает |
| 6.3.5 | Edit charge из закрытого периода (оба маршрута) | ✅ Скрыт |
| 6.3.6 | Бот `bot_close_period` | Не запускался |

---

## §7. Внебрифинговые правки — итог

| Таблица | Что изменено | Инициатор |
|---------|-------------|-----------|
| `ref_periods` | Update mode → Read-Only | Стратег |
| `charges` | Delete → Only if + slice Read-Only убран | Стратег |
| `payments` | Edit + Delete → Only if; Delete Position → Prominent; slice Read-Only убран | Стратег + владелец |
| `expenses` | Valid_if period исправлен (убрано -3); Edit + Delete → Only if; slice Read-Only убран | Стратег + владелец |

---

## §8. Замеченные смежные хвосты

| # | Хвост | Приоритет |
|---|-------|-----------|
| Х-1 | В `charges[period]` Valid_if ранее содержал `- 3` (разрешал ретро). Аналогично обнаружено и исправлено в `expenses[period]`. Проверить, нет ли аналогичного в других таблицах с Ref на `ref_periods`. | Средний |
| Х-2 | Паттерн «slice Read-Only блокирует actions» обнаружен на трёх slice-ах. Проверить остальные slice-а проекта. | Средний |
| Х-3 | Добавить паттерн slice Read-Only в appsheet-setup-SKILL.md. | Низкий |

---

## §9. Итог

**Изменено:**
- 1 лист `tariffs` в Google Sheets — удалён столбец `billing_frequency`.
- Schema AppSheet регенерирована.
- 1 колонка `charges[period]` — Initial value + Valid_if (текущий + 3 вперёд).
- `ref_periods` — Update mode = Read-Only.
- `charges` Delete — Only if condition добавлен.
- `charges_filtered` slice — Update mode: включены Updates + Deletes.
- `payments` Edit + Delete — Only if condition добавлен; Delete Position → Prominent.
- `payments_filtered` slice — Update mode: включены Updates + Deletes.
- `expenses[period]` Valid_if — исправлена нижняя граница (убрано -3).
- `expenses` Edit + Delete — Only if condition добавлен.
- `expenses_filtered` slice — Update mode: включены Updates + Deletes.

**Создано:** 0.
**Удалено:** 1 столбец в Sheets (`billing_frequency`).

**Версия AppSheet обновилась** с 1.000523 до 1.000535 в ходе сессии — без влияния на результат.

---

*Отчёт составил: Sonnet (без Adaptive thinking), 2026-05-11.*
