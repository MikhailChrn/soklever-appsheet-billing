# ARCHITECTURE.md — СО Клевер Биллинг

**Версия:** 0.2 (09.05.2026)
**Назначение документа:** долгоживущая память о технической реализации. Реальные составы таблиц, дословные формулы, цепочки actions и bots, накопленные best practices, карта подводных камней AppSheet.

**Что НЕ в этом документе:** бизнес-модель и принципы (`SYSTEM.md`), метафоры и термины (`GLOSSARY.md`), регламенты работы (`REGULATIONS.md`).

**Когда обновлять:** при добавлении/удалении колонок, изменении формул, появлении новых actions/bots/views, после каждого закрытого брифинга, затрагивающего архитектуру.

**Принцип:** дословные цитаты из живого AppSheet проекта, не пересказы. Если что-то описано «по памяти» — явно помечать.

---

## §1. Стек

### §1.1. Хранилище данных

**Google Sheets:** `2026-04-17 soklever-google-ancestor-v6 (main)` — рабочий файл. Имя историческое (последняя синхронизация эталонов с LLM была 17.04, главное — `(main)` в скобках). Не путать с бэкапами `soklever-google-ancestor-vN`.

**Состав листов:** 13 на момент 08.05.2026:
- 8 рабочих: `families`, `students`, `tariffs`, `student_tariffs`, `charges`, `payments`, `expenses`, `opening_balances`.
- 3 служебных: `settings`, `expense_categories`, `users`.
- 2 справочника: `ref_periods`, `ref_classes`.
- 2 рудимента, не подключены к AppSheet: `debts`, `dashboard` (к удалению, см. §6).

### §1.2. Приложение

**AppSheet:** `soklever-root-billing v.1.0`. Версия на 09.05.2026 — **1.000483** (после применения pre-#08 и pre-#08.1).

### §1.3. Логика

Вся серверная логика — внутри AppSheet (Actions, Bots, VC). **Apps Script не используется.** Закрытие периода — через AppSheet Bot, триггер — служебная колонка `settings[last_close_trigger]`.

---

## §2. Таблицы — реальные составы

Все составы — буквально как в Data view AppSheet на 06.05.2026 (v10) с уточнениями на 08.05.

### §2.1. `students` (21 колонка)

| # | Колонка | Тип | Заметка |
|---|---|---|---|
| 1 | `_RowNumber` | Number | системная |
| 2 | `student_id` | Text | KEY |
| 3 | `family_id` | Ref | → families |
| 4 | `first_name` | Name | |
| 5 | `last_name` | Name | |
| 6 | `class_group` | Ref | → ref_classes (после #6). Очищается при выбытии (pre-#08.1) |
| 7 | `status` | Enum | значения: «Активен», «Выбыл». Allow other = false. Initial Value = «Активен» (pre-#08) |
| 8 | `start_date` | Date | используется (pre-#08 — добавлено в форму создания) |
| 9 | `end_date` | Date | заполняется action «Выбытие» = `[withdrawal_date]` (pre-#08.1) |
| 10 | `withdrawal_date` | Date | **NEW в pre-#08.1.** Служебная: хранит дату выбытия из формы до применения транзакции. `Show? = false`, Display name «дата выбытия» |
| 11 | `birth_date` | Date | **NEW в pre-#08.1.** Дата рождения. `Require? = false`, Display name «дата рождения» |
| 12 | `comment` | Text | |
| 13 | `created_at` | DateTime | Display name «создан» (унифицировано в pre-#08.1) |
| 14 | `_ComputedName` | Name (VC, LABEL) | формула в §3.5. Header column в `students_Detail` — отображается всегда (см. §6.8) |
| 15 | `Related opening_balance` | List (VC) | `REF_ROWS("opening_balances", ...)` |
| 16 | `Related payments` | List (VC) | `REF_ROWS("payments", ...)` |
| 17 | `Related student_tariffs` | List (VC) | `REF_ROWS("student_tariffs", ...)`. Очищаются при выбытии (pre-#08.1) |
| 18 | `debt` | Number (VC) | формула в §3.3 |
| 19 | `Related charges` | List (VC) | `REF_ROWS("charges", ...)` |
| 20 | `overdue_debt` | Number (VC) | формула в §3.4 |
| 21 | `_form_header_students` | Show | косметика формы |

**Изменения относительно v10:** в v9/v10 `start_date`/`end_date` числились рудиментами. В pre-#08/pre-#08.1 они активированы (форма «Новый ученик» + кнопка «Выбытие»). К удалению **не идут**. Добавлены `withdrawal_date` (служебная) и `birth_date` (доменная).

**Slices на `students`:**
- `students_visible` (pre-#08.1) — Source: students, Row filter: `OR([status]="Активен", [debt]<>0)`, all columns, Updates+Adds+Deletes. Используется в основном view списка учеников.
- `students_withdrawal` (pre-#08.1) — Source: students, Columns: `student_id` (Key) + `withdrawal_date`, Update mode: Updates only. Используется только в `students_withdrawal_form`.

### §2.2. `families` (11 колонок)

| # | Колонка | Тип |
|---|---|---|
| 1 | `_RowNumber` | Number |
| 2 | `family_id` | Text (KEY) |
| 3 | `family_name` | Name (LABEL) |
| 4 | `phone` | Phone |
| 5 | `yandex_account` | Email |
| 6 | `comment` | Text |
| 7 | `created_at` | DateTime |
| 8 | `Related students` | List (VC) |
| 9 | `family_debt` | Number (VC) — формула в §3.3 |
| 10 | `family_overdue_debt` | Number (VC) — формула в §3.4 |
| 11 | `_form_header_families` | Show |

### §2.3. `student_tariffs` (8 колонок)

| # | Колонка | Тип | Заметка |
|---|---|---|---|
| 1 | `_RowNumber` | Number | |
| 2 | `assignment_id` | Text (KEY) | |
| 3 | `student_id` | Ref | → students |
| 4 | `tariff_id` | Ref | → tariffs |
| 5 | `comment` | Text | |
| 6 | `tariff_amount` | Number (VC) | App formula: `[tariff_id].[amount]`. Просто отображение, не индивидуальная сумма. |
| 7 | `_form_header_tariff` | Show | |
| 8 | `_student_name` | Name (LABEL, VC) | App formula: `[student_id].[_ComputedName]` |

### §2.4. `tariffs` (10 колонок)

| # | Колонка | Тип | Заметка |
|---|---|---|---|
| 1 | `_RowNumber` | Number | |
| 2 | `tariff_id` | Text (KEY) | |
| 3 | `tariff_name` | Name (LABEL) | |
| 4 | `tariff_type` | Enum | значения сейчас: «Регулярный». Используется в фильтрах. **Не рудимент.** См. §5.3. |
| 5 | `amount` | Number | |
| 6 | `billing_frequency` | Enum | бывшее `period`, переименовано в #6 |
| 7 | `comment` | LongText | |
| 8 | `Related student_tariffs` | List (VC) | |
| 9 | `Related charges` | List (VC) | |
| 10 | `_form_header_tariffs` | Show | |

**Состав строк на 08.05.2026** (9 тарифов, проверено в Sheets 05.05):

| ID | Название | tariff_type | amount | billing_frequency |
|---|---|---|---|---|
| T001 | 3 класс, база | Регулярный | 15000 | Месяц |
| T002 | 5 класс, база | Регулярный | 17000 | Месяц |
| T003 | 2 класс, база | Регулярный | 14000 | Месяц |
| T004 | 7 класс, база | Регулярный | 18000 | Месяц |
| T005 | Продлёнка Мини (до 5 дней) | Регулярный | 1000 | Месяц |
| T006 | Продлёнка Стандарт (до 10 дней) | Регулярный | 1500 | Месяц |
| T007 | Продлёнка Оптимал (до 14 дней) | Регулярный | 2000 | Месяц |
| T008 | Продлёнка Полный (20 дней) | Регулярный | 3500 | Месяц |
| T009 | Летний взнос (ремонт+аренда) | Регулярный | 5000 | Разовый ⚠️ семантика — см. §6 хвост |

**T010 «Летний тариф»** — будет создан в брифинге #08.

### §2.5. `charges` (10 колонок)

| # | Колонка | Тип | Заметка |
|---|---|---|---|
| 1 | `_RowNumber` | Number | |
| 2 | `charge_id` | Text (KEY) | |
| 3 | `student_id` | Ref → students | |
| 4 | `period` | Ref → ref_periods | (после #6) |
| 5 | `tariff_id` | Ref → tariffs | |
| 6 | `amount` | Number | обязательная |
| 7 | `charge_type` | Enum | значения: «Авто», «Ручное» (после #07.3, «Корректировка» удалена). Show=false на колонке (скрыта из форм). Initial Value = `"Ручное"`. |
| 8 | `description` | Text | |
| 9 | `created_at` | DateTime | Initial Value = `NOW()`. |
| 10 | `_form_header_charges` | Show | |

### §2.6. `payments`

Точный состав (TODO досверить дословно в следующей сессии):
```
payments: payment_id | student_id | amount | payment_date | comment | class_group (Spreadsheet formula)
```

**Важно (Р-1):** у `payments` **нет** поля `period`. Платежи не привязаны к месяцу — это часть модели сосудов.

`class_group` в `payments` — Text со Spreadsheet formula (VLOOKUP в Sheets из `students[class_group]`). Сознательная денормализация. AppSheet-формула не используется.

### §2.7. `expenses`

```
expenses: expense_id | amount | period→ref_periods | expense_date | category_id | description | created_by
```

**Р-2:** для дашборда и аналитики используется `period`, не `expense_date`.

`expenses[period_num]` — удалена в #6.

### §2.8. `expense_categories`

Простой справочник (`category_id`, `category_name`).

### §2.9. `opening_balances` (6 колонок)

| # | Колонка | Тип |
|---|---|---|
| 1 | `_RowNumber` | Number |
| 2 | `balance_id` | Text (KEY, LABEL) |
| 3 | `student_id` | Ref → students |
| 4 | `balance_date` | Date |
| 5 | `amount` | Number |
| 6 | `comment` | Text |

Используется в формулах `students[debt]` и `students[overdue_debt]` — вычитается. Если таблица пустая, вычитается ноль.

**Судьба** — в `SYSTEM.md` §8 как открытый вопрос.

### §2.10. `settings`

Однострочная таблица настроек. Колонки на 06.05.2026 (TODO досверить дословно):

- `id` — Text, KEY. Единственная строка.
- `current_period` — Ref → ref_periods, VC. Формула в §3.6.
- `academic_year_start` — Date, физическая. Заполняется вручную в Sheets, скрыта от UI. Скорее всего `2025-09-01`.
- `last_close_trigger` — DateTime, физическая, Show=false. Служебная для Data Change Event Bot закрытия периода.
- `last_close_at` — DateTime, физическая. Обновляется ботом. Отображается в плитке «Текущее закрытие».
- `last_close_message` — LongText, физическая. Текст последнего закрытия.
- `filter_period` — Ref → ref_periods. Фильтр архива.
- `filter_charges_period` — Ref → ref_periods.
- `filter_payments_period` — Ref → ref_periods.
- `filter_charges_class` — Ref → ref_classes.
- `filter_payments_class` — Ref → ref_classes.
- 9 VC дашборда (см. §3.1, §3.2) + хелпер `dash_пред_период`.

### §2.11. `ref_periods`

Справочник периодов (введён в #6). Одна колонка — `period`, **тип Text** (не Number, иначе AppSheet форматирует с разделителем `202 603`, см. BP-36).

Диапазон: `202601` … `203012` (60 строк, 5 лет). Заполнен вручную протягиванием. Без атрибутов. **Без автогенерации Bot'ом.**

### §2.12. `ref_classes`

Справочник классов (введён в #6). Одна колонка — `class_group`, тип Text. 12 строк: «0 класс» … «11 класс». Без атрибутов.

### §2.13. `users`

`email` (Text), `role` (Enum: «ADMIN», «OPERATOR»). Используется в Show_If типа `LOOKUP(USEREMAIL(), "users", "email", "role") = "ADMIN"`.

### §2.14. `debts`

**Мёртвая** таблица в AppSheet. По Р-5 не используется. К удалению (см. §6).

### §2.15. Листы-рудименты в Sheets

`dashboard`, `debts` — рудименты эпохи до AppSheet. Не подключены к приложению. К удалению.

---

## §3. Формулы — дословно

Все формулы — копия из живого проекта на 06.05.2026, если не указано иное.

### §3.1. Дашборд — Живая зона (открытый период)

**`settings[dash_открытый_поступления]`** (Number, VC):
```
SUM(SELECT(payments[amount],
  AND(
    YEAR([payment_date]) = NUMBER(LEFT(ANY(settings[current_period]), 4)),
    MONTH([payment_date]) = NUMBER(RIGHT(ANY(settings[current_period]), 2))
  )
))
```

**`settings[dash_открытый_расходы]`** (Number, VC):
```
SUM(SELECT(expenses[amount],
  NUMBER([period]) = NUMBER(ANY(settings[current_period]))
))
```

**`settings[dash_открытый_начислено]`** (Number, VC):
```
SUM(SELECT(charges[amount],
  NUMBER([period]) = NUMBER(ANY(settings[current_period]))
))
```

**Архитектурное наблюдение:** три формулы используют **разные** способы определить «текущий период»:
- Поступления — по `payments[payment_date]` через `YEAR()`+`MONTH()`. Потому что у `payments` нет `period` (Р-1).
- Расходы и начисления — по `[period] = current_period`. У `expenses` и `charges` `period` есть.

Это **намеренно**. Платёж бэк-датой за март попадает в мартовские поступления, не в апрельские.

### §3.2. Дашборд — Долги и Твёрдая земля

**`settings[dash_долги_количество]`** (Number, VC):
```
COUNT(SELECT(families[family_id], [family_overdue_debt] > 0))
```

⚠️ **Открытый вопрос v10 §7.2:** в #07.3 в NO-сообщении бота это поле дало `1`, плитка «Долги» показывала `3`. Гипотеза: формула считается на pre-commit состоянии Bot — VC ещё не пересчитались после ForEach.

**`settings[dash_долги_сумма]`** (Number, VC):
```
SUM(SELECT(families[family_overdue_debt], [family_overdue_debt] > 0))
```

**`settings[dash_земля_получено]`** (Number, VC):
```
SUM(SELECT(payments[amount],
  AND(
    [payment_date] >= ANY(settings[academic_year_start]),
    OR(
      YEAR([payment_date]) < NUMBER(LEFT(ANY(settings[current_period]), 4)),
      AND(
        YEAR([payment_date]) = NUMBER(LEFT(ANY(settings[current_period]), 4)),
        MONTH([payment_date]) < NUMBER(RIGHT(ANY(settings[current_period]), 2))
      )
    )
  )
))
```

**`settings[dash_земля_израсходовано]`** (Number, VC):
```
SUM(SELECT(expenses[amount],
  AND(
    NUMBER([period]) >= YEAR(ANY(settings[academic_year_start])) * 100 + MONTH(ANY(settings[academic_year_start])),
    NUMBER([period]) < NUMBER(ANY(settings[current_period]))
  )
))
```

**`settings[dash_земля_свободные]`** (Number, VC):
```
[dash_земля_получено] - [dash_земля_израсходовано]
```

**`settings[dash_пред_период]`** (Number, VC) — хелпер:
```
IFS(
  NUMBER(RIGHT(ANY(settings[current_period]), 2)) > 1,
  NUMBER(LEFT(ANY(settings[current_period]), 4)) * 100 + NUMBER(RIGHT(ANY(settings[current_period]), 2)) - 1,
  TRUE,
  (NUMBER(LEFT(ANY(settings[current_period]), 4)) - 1) * 100 + 12
)
```

Возвращает YYYYMM предыдущего месяца с корректным переходом через границу года.

### §3.3. Текущая задолженность

**`students[debt]`** (Number, VC) — баланс сосуда ученика:
```
SUM(SELECT(charges[amount], [student_id] = [_THISROW].[student_id]))
- SUM(SELECT(payments[amount], [student_id] = [_THISROW].[student_id]))
- SUM(SELECT(opening_balances[amount], [student_id] = [_THISROW].[student_id]))
```

Может быть положительным, нулевым, отрицательным.

**`families[family_debt]`** (Number, VC):
```
SUM(SELECT(students[debt], [family_id] = [_THISROW].[family_id]))
```

**Без обрезки нулём.** Переплата одного ученика **компенсирует** долг другого внутри семьи.

### §3.4. Дебиторская задолженность (просрочка)

**`students[overdue_debt]`** (Number, VC) — только просроченная часть:
```
IFS(
  SUM(SELECT(charges[amount],
    AND(
      [student_id] = [_THISROW].[student_id],
      NUMBER([period]) < NUMBER(ANY(settings[current_period]))
    )
  ))
  - SUM(SELECT(payments[amount], [student_id] = [_THISROW].[student_id]))
  - SUM(SELECT(opening_balances[amount], [student_id] = [_THISROW].[student_id]))
  > 0,
  SUM(SELECT(charges[amount],
    AND(
      [student_id] = [_THISROW].[student_id],
      NUMBER([period]) < NUMBER(ANY(settings[current_period]))
    )
  ))
  - SUM(SELECT(payments[amount], [student_id] = [_THISROW].[student_id]))
  - SUM(SELECT(opening_balances[amount], [student_id] = [_THISROW].[student_id])),
  TRUE,
  0
)
```

Эквивалент `MAX(0, ...)`, написанный длинно через IFS. Выражение продублировано — **при правке менять в обоих местах**. Кандидат на упрощение через `MAX(0, ...)` в будущем.

**`families[family_overdue_debt]`** (Number, VC):
```
SUM(SELECT(students[overdue_debt], [family_id] = [_THISROW].[family_id]))
```

**Каждый `overdue_debt` уже обрезан нулём.** Переплата одного ученика **не** компенсирует просрочку другого.

**Намеренная асимметрия с `family_debt`** (подтверждено владельцем 03.05.2026): для текущего баланса семья = единый сосуд, для просрочки — каждый ученик отдельно.

### §3.5. _ComputedName

**`students[_ComputedName]`** (Name, VC, LABEL):
```
CONCATENATE([first_name], " ", [last_name], " (", [class_group], ")")
```

После #6 `class_group` — Ref на `ref_classes`. CONCATENATE с Ref-полем выдаёт значение ключа («3 класс»).

### §3.6. current_period

**`settings[current_period]`** (Ref → ref_periods, **физическая колонка** с 13.05.2026):

**Хранение:** столбец N в листе `settings` Google Sheets.

**Семантика (с 13.05.2026 #16, вариант C+):** «открытый период» — тот, в который сейчас идут регулярные начисления. То есть в нормальном ходе совпадает с `max(charges[period])`.

**Обновление:** action `settings__update_trigger` пишет **«следующий период от max(charges)»**:
```
INDEX(SORT(SELECT(ref_periods[period],
  NUMBER([period]) > NUMBER(INDEX(SORT(SELECT(charges[period], TRUE), TRUE), 1))
)), 1)
```
одновременно с `last_close_trigger = NOW()`. В момент клика «Закрыть период» физическая колонка обновляется значением «следующий период по справочнику от текущего max(charges[period])», и **только потом** ForEach создаёт charges за этот период.

**Почему так — «вариант C+» от Gemini.** До 13.05.2026 в `settings__update_trigger` стоял прямой max charges (`INDEX(SORT(charges[period], TRUE), 1)`). Это давало правильное значение **до** запуска бота. Но `student_tariffs__create_next_charge` внутри ForEach считал «period + 1» через справочник от `settings[current_period]`. Получалось: settings писал старое max (202605), ForEach создавал новые charges (202606), `current_period` оставался 202605. После коммита транзакции — рассинхрон между `settings` (202605) и реальностью в charges (202606).

Решение: **сначала записать целевое состояние** в settings (открытый период = next от max), потом ForEach создаёт charges за это значение напрямую (`ANY(settings[current_period])`). После коммита `settings[current_period]` совпадает с max(charges[period]) — согласованность.

**Почему физическая, а не VC** (фундаментальный паттерн AppSheet):

До 13.05.2026 это была VC. Обнаружено в #15: **внутри AppSheet bot transaction VC читаются из замороженного snapshot всех таблиц**, не пересчитываются. Если VC зависит от другой таблицы (`current_period` зависит от `charges`), snapshot может быть устаревшим — бот видит значение, актуальное на момент **предыдущей** активности.

Симптом #15: `bot_close_period` создавал charges за 202604, при том что VC `current_period` на frontend и Data Explorer показывала 202605. Подтверждено в Monitor: в Output Input-шага одна VC `current_period` показывала 202605, другая VC `dash_пред_период` (= current - 1) показывала 202602. Несогласованность внутри одной транзакции = разные моменты «заморозки».

Подтверждено внешним консультантом Gemini: классическая особенность AppSheet bot snapshot, рекомендация — перевод в физическую колонку + логика «обновить до работы».

**`bot_summer_apply` `current_period` НЕ обновляет** — он не меняет periods в charges. Если в будущем летний перевод начнёт создавать charges (например, июньские сразу при переключении) — нужно будет добавить обновление и в `settings__set_summer_trigger`.

**Ограничение варианта C+:** ручное удаление строк в Sheets `charges` **не пересчитывает** `settings[current_period]` автоматически — никакой action не реагирует. `current_period` синхронизируется только при следующем запуске `bot_close_period`. Это осознанное решение (§7.1 SYSTEM.md): прямая правка Sheets — аварийный канал, не штатный поток. При необходимости владелец синхронизирует вручную.

### §3.7. opening_balances в формулах долга

Обе формулы (§3.3, §3.4) **вычитают** `SUM(opening_balances[amount])` для ученика. Если таблица пустая — вычитается ноль.

### §3.8. payments без фильтра по дате

В формулах `debt` и `overdue_debt` платежи учитываются **все**, без фильтра. Правильно по модели сосудов.

### §3.9. Valid_If на Ref-колонках периода

Окно вокруг `current_period` (применено к `charges[period]`, `expenses[period]`, фильтрам):
```
SELECT(
  ref_periods[period],
  AND(
    NUMBER([period]) >= NUMBER(ANY(settings[current_period])) - 3,
    NUMBER([period]) <= NUMBER(ANY(settings[current_period])) + 3
  )
)
```

На границе года формула даёт усечённое окно (например, в декабре `[202609..202612]` вместо `[202609..202703]`) — приемлемо.

Для классовых фильтров: `SELECT(ref_classes[class_group], TRUE)`.

### §3.10. Initial Value на period-колонках

```
ANY(settings[current_period])
```

Применено к `charges[period]` и `expenses[period]`.

### §3.11. last_close_at, last_close_message (NEW в #07.3, скорректированы в #11)

**`settings[last_close_at]`** — DateTime, физическая. Обновляется ботом в шагах `step_yes_already_closed` и `step_no_set_result` через `Set row values`. Значение = `NOW()`.

**`settings[last_close_message]`** — LongText, физическая.

В **YES-ветке** (`step_yes_already_closed`):
```
CONCATENATE("Начисления за текущий период (", ANY(settings[current_period]), ") уже существуют!")
```

В **NO-ветке** (`step_no_set_result`) — формула после фикса #16 (13.05.2026):
```
CONCATENATE(
  "Период ",
  INDEX(SORT(SELECT(ref_periods[period],
    NUMBER([period]) < NUMBER(ANY(settings[current_period]))
  ), TRUE), 1),
  " закрыт. Открытый период: ",
  ANY(settings[current_period]),
  ". Создано начислений: ",
  COUNT(SELECT(student_tariffs[assignment_id], [student_id].[status] = "Активен")),
  "."
)
```

**Почему так:**
- «Закрытый» = max период из `ref_periods` строго меньше текущего открытого. Не читаем `charges` — snapshot уже содержит свежесозданные строки за новый период, дал бы +1 (косметический баг v15, см. §8.4 SYSTEM.md).
- «Открытый» = `ANY(settings[current_period])`. Шаг 1 уже записал сюда новое значение (next от max), оно гарантированно актуально на этом шаге.
- «Создано» = количество активных student_tariffs. Считать через `SELECT(charges[charge_id], [period] = current_period)` нельзя по той же причине — snapshot в финальных шагах целевую таблицу видит частично.

### §3.12. Action chain «Выбытие ученика» (NEW в pre-#08.1)

**Проблема:** «Выбытие» = транзакция из 4 операций в двух таблицах: `students` (`end_date`, `status`, `class_group`) + `student_tariffs` (delete всех строк ученика). Плюс UX-требование — ручной выбор даты пользователем.

**Путь 1 (Action with Inputs)** не сработал: секция Inputs **отсутствует** в AppSheet 1.000444. Обнаружено в pre-#08.1, см. §6.9.

**Путь 2 — реализован.** Архитектура — Slice + Form view + LINKTOROW + Form Saved + Grouped action.

**Артефакты:**

| # | Артефакт | Описание |
|---|---|---|
| 1 | Колонка `students[withdrawal_date]` | Date, `Show? = false`. Служебная — буфер даты до применения транзакции. |
| 2 | Slice `students_withdrawal` | Source: students, Columns: `student_id` (Key) + `withdrawal_date`, Update mode: **Updates only** (не Adds — критично, иначе появляется кнопка «+»). |
| 3 | Form view `students_withdrawal_form` | Position: ref, For data: `students_withdrawal`, Column order Manual: только `withdrawal_date`. Display name: «Дата выбытия». **Form Saved** event → action `students__withdrawal_apply`. |
| 4 | Action `students__withdrawal` | **Видимый**, на таблице `students`. Тип: `App: go to another view within this app`. Target: `LINKTOROW([_THISROW], "students_withdrawal_form")`. Position: Prominent. Display name: «выбытие». Confirmation: `TRUE`. Confirmation Message: `CONCATENATE("Вы уверены, что ученик ", [first_name], " ", [last_name], " заканчивает обучение в периоде ", ANY(settings[current_period]), "?")`. Show_If: `AND([status]="Активен", LOOKUP(USEREMAIL(),"users","email","role")="ADMIN")`. |
| 5 | Action `students__withdrawal_apply` | **Скрытый** (Position: Hide), Grouped. Содержит шаги: (1) `students__withdrawal_set_fields`, (2) `students__delete_tariffs`. |
| 6 | Action `students__withdrawal_set_fields` | **Скрытый**, на `students`. Тип: `Data: set the values of some columns in this row`. Sets: `end_date = [withdrawal_date]`, `status = "Выбыл"`, `class_group = ""`. |
| 7 | Action `students__delete_tariffs` | **Скрытый**, на `students`. Тип: `Data: execute an action on a set of rows`. Referenced table: `student_tariffs`. Referenced rows: `SELECT(student_tariffs[assignment_id], [student_id]=[_THISROW].[student_id])`. Referenced action: `student_tariffs__delete_one`. |
| 8 | Action `student_tariffs__delete_one` | **Скрытый**, на `student_tariffs`. Тип: `Data: delete this row`. Вспомогательный — единичное удаление, вызывается из (7). |

**UX-поток:**
1. Пользователь нажимает «выбытие» в карточке ученика → Confirmation с именем и периодом.
2. Подтверждает → AppSheet открывает `students_withdrawal_form` через `LINKTOROW` (форма в режиме редактирования существующей строки).
3. Пользователь выбирает дату → Save.
4. Form Saved триггерит Grouped `students__withdrawal_apply`:
   - Set: `end_date=[withdrawal_date]`, `status="Выбыл"`, `class_group=""`.
   - Delete: все `student_tariffs` ученика.

**Почему `LINKTOROW`, а не `LINKTOFORM`:** см. §6.9 и BP-47.

### §3.13. Прочие actions UX-уровня (pre-#08, pre-#08.1)

**`families__add_student`** (pre-#08, на таблице `families`):
- Тип: `App: go to another view within this app`.
- Target: `LINKTOFORM("students_Form", "family_id", [family_id])`.
- Position: Prominent. Display name: «добавить ребёнка».
- Show_If: `LOOKUP(USEREMAIL(),"users","email","role")="ADMIN"`.
- Открывает форму создания ученика с предзаполненной семьёй. **Здесь `LINKTOFORM` правомерен — создаётся новая запись.**

**Форма `students_Form`** (системная, настроена в pre-#08):
- Column order: Manual — `_form_header_students`, `first_name`, `last_name`, `family_id`, `class_group`, `start_date`, `birth_date` (NEW в pre-#08.1), `comment`.
- `status` и `end_date` **не включены** в Column order (скрыты при создании, видны только в Detail view существующих учеников через `Show_If = ISNOTBLANK([student_id])`).
- `students[status]` Initial Value = `"Активен"`.

**Унификация Detail view** (pre-#08.1):
- Все `*_id` колонки (`charge_id`, `expense_id`, `payment_id`) — `Show? = false`.
- Все `created_at` — Display name = «создан».
- `expenses[period]` — Display name = «период». Delete action — Position: Prominent (для исправления ошибочных расходов).
- `payments`: `class_group`, `_ComputedKey` — `Show? = false`.

**Очистка view-фильтров** (pre-#08.1):
- `settings → action Delete` на блоке фильтра — Position: Hide.
- `payments_filtered`, `expenses_filtered` Slices — Update mode: Read-Only (был Updates+Adds+Deletes).
- `payments_filtered_Form` view — удалён (конфликт: form view на Read-Only slice).

---

## §4. Closure architecture (закрытие периода)

### §4.1. Цель

Кнопка «Закрыть период» в фрейме «скрипты». По клику: подтверждение → массовая генерация регулярных начислений на следующий месяц для активных учеников → запись результата в `settings[last_close_at]` + `settings[last_close_message]` → отображение на дашборде. Защита от повторного нажатия в тот же календарный месяц.

### §4.2. Цепочка вызова

1. Пользователь на view «скрипты» нажимает кнопку «Закрыть период».
2. Срабатывает Action `settings__close_period_trigger` (`Data: execute an action on a set of rows`).
3. `settings__update_trigger` пишет `NOW()` в `settings[last_close_trigger]`.
4. Bot `bot_close_period` ловит изменение через Event с условием `[_THISROW_BEFORE].[last_close_trigger] <> [_THISROW_AFTER].[last_close_trigger]`.
5. Bot выполняет Branch:
   - **YES (закрытие уже произошло)** → `Set row values` записывает `last_close_at` и `last_close_message` с текстом «Начисления за текущий период уже существуют».
   - **NO** → ForEach по `student_tariffs` (фильтр: активные + регулярный тариф) → `student_tariffs__create_next_charge` создаёт `charges` → `Set row values` записывает результат с COUNT.

### §4.3. Action `settings__update_trigger`

- For a record of this table: `settings`
- Do this: `Data: set the values of some columns in this row`
- Set columns:
  - `last_close_trigger` = `NOW()`
  - `current_period` = `INDEX(SORT(SELECT(ref_periods[period], NUMBER([period]) > NUMBER(INDEX(SORT(SELECT(charges[period], TRUE), TRUE), 1)))), 1)` (с 13.05.2026 #16, вариант C+, см. §3.6)

### §4.4. Action `settings__close_period_trigger`

- For a record of this table: `settings`
- Do this: `Data: execute an action on a set of rows`
- Referenced Table: `settings`
- Referenced Rows: `SELECT(settings[id], TRUE)` (или эквивалент)
- Referenced Action: `settings__update_trigger`
- Display name: «Закрыть период»
- Show_If: `AND(LOOKUP(USEREMAIL(), "users", "email", "role") = "ADMIN", CONTEXT("View") = "скрипты")`
- Needs confirmation: ON
- Confirmation Message (через `CONCATENATE` — синтаксис `<<...>>` не работает): «Вы точно хотите закрыть период YYYYMM? Следующий открытый период будет YYYY(MM+1).»
- Position: Prominently.

### §4.5. Action `student_tariffs__create_next_charge`

- For a record of this table: `student_tariffs`
- Do this: `Data: add a new row to another table using values from this row`
- Table to add to: `charges`
- Set columns:
  - `charge_id` = `UNIQUEID()`
  - `student_id` = `[student_id]`
  - `tariff_id` = `[tariff_id]`
  - `charge_type` = `"Авто"`
  - `period` = `ANY(settings[current_period])` (с 13.05.2026 #16, вариант C+, см. §3.6: открытый период уже записан Шагом 1)
  - `amount` = `[tariff_id].[amount]`
  - `description` = `""`
- Show_If: `FALSE`.

### §4.6. Bot `bot_close_period`

**Event `event_close_period_trigger`:**
- Source table: `settings`
- Update events: Updates only
- Condition: `[_THISROW_BEFORE].[last_close_trigger] <> [_THISROW_AFTER].[last_close_trigger]`

**Process:**

```
event_close_period_trigger
        ↓
   step_branch_check  (Branch)
       ↙           ↘
     YES            NO
      ↓              ↓
step_yes_           step_no_
already_closed      create_charges (ForEach)
(Set row values)        ↓
      ↓             step_no_set_result
     END            (Set row values)
                        ↓
                       END
```

Каждая ветка завершается самостоятельно — общего финала нет.

### §4.7. step_branch_check — формула

После фикса 06.05 (обнаружен инцидент каскадирования, см. §5):
```
AND(
  NUMBER(ANY(settings[current_period])) > (YEAR(TODAY()) * 100) + MONTH(TODAY()),
  ISNOTBLANK(SELECT(charges[charge_id], AND(
    [period] = ANY(settings[current_period]),
    [charge_type] = "Авто"
  )))
)
```

**Семантика:** Branch=YES (закрытие уже произошло), если **оба** условия:
1. `current_period` опередил календарный месяц.
2. За `current_period` есть начисления `charge_type = "Авто"`.

Календарная привязка защищает от каскадирования. Условие 2 защищает от аномалий с ручной подстановкой.

### §4.8. step_yes_already_closed

- Тип: `Set row values`.
- Table: `settings`.
- Set columns: `last_close_at` = `NOW()`, `last_close_message` = (см. §3.11).

### §4.9. step_no_create_charges (ForEach)

- Тип: `Run actions on rows`.
- Referenced Table: `student_tariffs`.
- Referenced rows:
  ```
  SELECT(student_tariffs[assignment_id],
    AND(
      [tariff_id].[tariff_type] = "Регулярный",
      [student_id].[status] = "Активен"
    )
  )
  ```
- Referenced Action: `student_tariffs__create_next_charge`.

### §4.10. step_no_set_result

- Тип: `Set row values`.
- Table: `settings`.
- Set columns: `last_close_at` = `NOW()`, `last_close_message` = (см. §3.11, NO-ветка после фикса #11).

### §4.11. UI — view «скрипты»

**MENU NAVIGATION:** пункт «скрипты» с иконкой молнии.
**View «скрипты»:** Detail view на `settings`. Show_If view = ADMIN.
Внутри показана кнопка `settings__close_period_trigger`.

**Системные actions Add/Edit/Delete** — оставлены (косметика, не критично, ADMIN-only).

**Хвост #07.3:** в попытке скрыть KEY-колонку `id` Sonnet упёрся в ограничение AppSheet (KEY на writable-таблице нельзя скрыть). Возможный обход: read-only Slice. Не критично.

### §4.12. Дашборд — плитка «Текущее закрытие»

**Slice:** `settings_dash_закрытие` на таблице `settings`. Колонки: `id`, `last_close_at`, `last_close_message`. Update mode: Read-Only.
**View:** `dash_закрытие`, тип Detail. Display name: «Текущее закрытие».
**Dashboard:** добавлена четвёртой плиткой в `главная`. Размер Large.

---

## §5. Архитектурные решения (Records)

Сводный список ADR. Полные обоснования — в исходных бэкапах (`subgeneral-backups/`).

| ID | Содержание | Источник |
|---|---|---|
| **Р-1** | У `payments` нет поля `period`. Платёж не привязан к месяцу. | v3, подтверждено #8 |
| **Р-2** | У `expenses` есть поле `period`. Дашборд использует его, не `expense_date`. | v3 |
| **Р-4** | `payment_date` редактируется (default = `TODAY()`). | v5 |
| **Р-5** | Динамический долг без истории. Таблица `debts` не используется (к удалению). | v3 |
| **Р-6** | Должник = `family_overdue_debt > 0` по периодам строго раньше `current_period`. | v5 |
| **Р-7** | Закрытие периода — в конце месяца (28-31), вручную кнопкой. | v5 |
| **Р-8** | Тарифы и инфляция — без истории. `tariffs[period]` переименован в `billing_frequency` в #6. Колонки `valid_from`/`valid_to` удалены в #4. | v5, дополнено в #6 |
| **Р-9** | Дашборд «Главная» — три блока: живая зона, долги, твёрдая земля. + плитка «Текущее закрытие» в #07.3. | v4, дополнено в #07.3 |
| **Р-11** | Массовая летняя смена тарифа. Дедлайн на скрипт: 01.06.2026. | v5 |
| **Р-12** | Цветовая индикация: синий — текущая задолженность, красный — просрочка. | v6 |
| **Р-14** | `student_tariffs` без дат, без истории. | v7 |
| **Р-15** | Регламент бэкапа перед опасными операциями. | v7, см. REGULATIONS.md |
| **Р-16** | Период и класс — справочники, не Enum. Закрытие ТД-1. | #8, реализовано в #6 |

### §5.1. §Р-16 (детально)

Принят 02.05.2026 на архсессии (`subgeneral-backups/2026-05-02_arch-session-td1.md`), реализован в брифинге #6.

**Постановка:** значения периодов и классов хранятся в **справочных таблицах** (`ref_periods`, `ref_classes`). Все колонки, ранее имевшие тип Enum, переведены в Ref на эти справочники. Рассинхронизация значений (`"11класс"` vs `"11 класс"`) становится физически невозможной.

**Колонки, переведённые в Ref:**

| Таблица.колонка | Ref на |
|---|---|
| `charges[period]` | `ref_periods` |
| `expenses[period]` | `ref_periods` |
| `settings[current_period]` | `ref_periods` |
| `settings[filter_period]` | `ref_periods` |
| `settings[filter_charges_period]` | `ref_periods` |
| `settings[filter_payments_period]` | `ref_periods` |
| `students[class_group]` | `ref_classes` |
| `settings[filter_charges_class]` | `ref_classes` |
| `settings[filter_payments_class]` | `ref_classes` |

**`payments[class_group]` — не трогалась.** Остаётся Text со Spreadsheet formula (VLOOKUP в Sheets из `students[class_group]`). Сознательная денормализация. Если когда-нибудь захочется поменять способ хранения класса в `students` — `payments[class_group]` сломается.

### §5.2. Принцип «работать без владельца»

Введён на архсессии #8. Любой человек, открывший проект через год, должен понимать архитектуру по самому проекту. Логика — в редакторе AppSheet (видно), не в скрытом Apps Script.

Следствия: справочники вместо Enum'ов («одно место правды»), не закладывать атрибуты «на будущее», не делать автогенерацию через Bot, не вводить движущиеся части без необходимости.

### §5.3. `tariff_type` vs `charge_type` — две разные оси

Введено в v10 §2.3 после поправки v9.

**`tariffs[tariff_type]`** — тип регулярности тарифа. Сейчас все 9 тарифов: «Регулярный». Используется в фильтрах:
- Bot ForEach (`step_no_create_charges`): отбирает регулярные тарифы для генерации charges.
- `settings[current_period]`: отбирает period из charges по регулярным тарифам.

**`charges[charge_type]`** — источник начисления. После #07.3 только два значения: «Авто» (бот) и «Ручное» (форма разового начисления). «Корректировка» удалена — теперь это «Ручное» с отрицательной суммой.

При написании формул не путать.

---

## §6. Технический долг и хвосты

### §6.1. ТД-3 — косметика карточки тарифа

`tariff_id` (T001) в шапке карточки тарифа. Если появится — Show? = false. Не блокер.

### §6.2. ТД-5 — чистка рудиментов

Сводный список к удалению (отдельный микро-брифинг после #8):

1. **`opening_balances`** — судьба под вопросом. Если в проде всегда пустая, она и ссылки в `students[debt]`, `students[overdue_debt]` могут быть удалены. **Порядок важен:** сначала вычистить `SUM(SELECT(opening_balances[amount], ...))` из обеих формул долга, потом таблицу.
2. **`debts`** (таблица в AppSheet). Тривиально удаляется.
3. **Лист `debts` в Sheets** — рудимент эпохи до AppSheet.
4. **Лист `dashboard` в Sheets** — то же.

**Когда:** после #8 (когда дедлайн 01.06.2026 закроется).

**NB:** в v9 в этот список ошибочно были включены `students[start_date]`/`end_date` и `tariffs[tariff_type]`. После v10/pre-#08 они **исключены** — все три используются.

### §6.3. T009 «Летний взнос» — semantic

`tariff_type = "Регулярный"`, `billing_frequency = "Разовый"` — расхождение. Сейчас никому не привязан через `student_tariffs`. Если кто-то привяжет — Bot будет начислять каждый месяц.

Решать в отдельном мини-брифинге, возможно вместе с #8.

### §6.4. View-фильтры

`фильтр начислений`, `фильтр платежей`, `фильтр расходов` — компромиссное решение времён, когда не знали про встроенный фильтр AppSheet. После pre-#08.1 (чистка лишних кнопок) и периода тестирования встроенного фильтра — рассмотреть удаление.

### §6.5. VC внутри Bot transaction (открытый вопрос v10 §7.2)

В шаге `Set row values` Bot обращение к VC даёт **pre-commit** состояние. Влияет на текст NO-сообщения (расхождение `dash_долги_количество = 1` в боте vs `3` на дашборде).

Решено в #11: убрать «Должников» из NO-сообщения, оставить только COUNT начислений.

### §6.6. Расхождение charges 5 vs 6 в сценарии 3 теста (v10 §7.3)

В тесте #07.3 первое закрытие создало 5 charges, второе — 6. Не блокер, но стоит понять. TODO для одной из следующих сессий.

### §6.7. Error-run `bot_close_period` 04.05 в 15:45:47 (v10 §7.5)

Один из 6 прогонов закончился с Error. Не расследован. Не блокер.

### §6.8. `_ComputedName` для выбывших учеников (NEW в pre-#08.1)

**Проблема:** в Detail view ученика `_ComputedName` отображается **всегда** как Header column. Формула (§3.5):
```
CONCATENATE([first_name], " ", [last_name], " (", [class_group], ")")
```

После выбытия `class_group` очищается → имя становится `Иванов Иван ()` (пустые скобки). Гипотеза стратега «закроется через `class_group=""`» **не подтвердилась** — внутри скобок остаётся пусто, скобки видны.

**Варианты решения** (требуют архитектурного выбора):
1. Переписать `_ComputedName` с условием: `IF([status]="Выбыл", CONCATENATE([first_name], " ", [last_name]), CONCATENATE([first_name], " ", [last_name], " (", [class_group], ")"))`.
2. Сменить Header column на `first_name` + `last_name` без класса.

Не блокер для запуска 01.06.2026. Решать после #8.

### §6.9. VC игнорируют Column order в Detail view (NEW в pre-#08.1)

**Поведение AppSheet:** виртуальные колонки (`debt`, `overdue_debt`, и др.) рендерятся в Detail view **после** физических, независимо от порядка в Column order Manual. Обход не найден.

**Последствие для проекта:** в карточке ученика `debt` и `overdue_debt` всегда внизу, рядом с Related-секциями. Эстетика страдает, функционально работает.

**Не блокер.** Принять как ограничение AppSheet.

### §6.10. Action with Inputs недоступен в текущей версии AppSheet (NEW в pre-#08.1)

**Контекст:** в брифе pre-#08.1 §4.1 предложен Путь 1 — Action with Inputs (свежая фича AppSheet). На версии 1.000444 секция Inputs **отсутствует** в редакторе actions. Реализован Путь 2 (Slice + Form view + LINKTOROW).

**Хвост:** при обновлении AppSheet до версии с Inputs — рассмотреть упрощение архитектуры выбытия (избавиться от служебной колонки `withdrawal_date`, Slice `students_withdrawal`, Form view `students_withdrawal_form`). Не блокер, чисто эстетика.

### §6.11. Предупреждение в `students_withdrawal_form` (NEW в pre-#08.1)

В форме выбытия AppSheet показывает информационное предупреждение: `Table students_withdrawal does not allow new entries`. Не мешает работе.

**Причина:** Slice `students_withdrawal` имеет Update mode = Updates only (без Adds). Это критично — иначе появится кнопка «+» в форме.

**Решение:** оставить как есть. Принципиально не лечится без ухудшения UX.

---

## §7. Best Practices (BP)

Базовые BP (BP-1, BP-17–BP-35) — без изменений с v7. Здесь только новые.

### §7.1. Из ветки #6/#8

**BP-36. Тип ключа справочника — Text, не Number.**
Если ключ — числовое значение (например, YYYYMM), всё равно объявлять Text. Number в AppSheet форматируется с разделителем тысяч (`202 603`) и в UI выглядит сломанным. Если нужны числовые сравнения — приводить через `NUMBER()` в формулах.

**BP-37. Valid_If на Ref-колонке фильтрует dropdown, если возвращает список ключей.**
Формат: `SELECT(ref_table[key_column], <условие>)`. Yes/No-формула не фильтрует dropdown, только валидирует после выбора. Источник: документация AppSheet «Drop-down on a Ref column».

**BP-38. После добавления новой таблицы AppSheet генерирует Detail/Form views и View Ref actions.**
Если таблица — справочник, эти артефакты надо удалить/скрыть:
- UX → Views → SYSTEM GENERATED → удалить `<table>_Detail`, `<table>_Form`.
- Behavior → Actions → скрыть `View Ref (<column>)` на родительских таблицах.

**BP-39. AppSheet может автоматически конвертировать тип смежных колонок при изменении источника.**
После перевода `charges[period]` в Ref на `ref_periods`, AppSheet автоматически проставил Ref на `settings[current_period]` (логично) и **некорректно** — на `settings[filter_charges_class]` и `settings[filter_payments_class]` с источником `ref_periods` вместо `ref_classes`. **Любую автоконвертацию проверять вручную.**

**BP-40. AppSheet падает на расхождении числа колонок Sheets vs схемы.**
«Колонкой» Sheets считает любой столбец с непустой формулой/форматированием. Если в Sheets остались побочные формулы (диагностика, проверки), AppSheet при следующем sync не загрузит таблицу — `mismatch in number of columns`. Лечение — очистка Sheets через `Delete column` (не Clear). **Regenerate Schema не требуется.** Версия приложения от очистки не двигается.

**BP-41. AppSheet App Documentation не выгружает скрытые колонки.**
Если у колонки `Show? = false`, она может не попасть в HTML-экспорт. При важных аудитах сверяться с Sheets напрямую.

**BP-42. После Save счётчик версий AppSheet растёт. Regenerate Schema и очистка Sheets — не двигают.**
Версия меняется только от изменений в редакторе AppSheet, сохранённых через Save.

### §7.2. Из ветки #7 (закрытие периода)

**BP-43. Имена Enum-значений сверять с проектом перед написанием формул.**
В сессии #9 спотыкались на «Авто» vs «Регулярный». Источник правды — Data view AppSheet, не догадки. Это пробивалось через всю сессию #9 и стоило часа на тестировании #7.

**BP-44. Формула Branch с проверкой «есть ли charges за следующий период» вызывает каскадирование.**
После каждого закрытия `current_period` сдвигается на этот «следующий», и проверка перестаёт срабатывать. Правильное решение — **календарная привязка** (см. §4.7) + проверка наличия начислений за `current_period`.

### §7.3. Из ветки pre-#08.1 (UX и actions)

**BP-45. `LINKTOFORM` всегда открывает форму создания, не редактирования.**
Передача KEY существующей строки **не помогает** — AppSheet выдаёт ошибку `There is already a row with the key: '<KEY>'`. Для редактирования существующей строки через форму использовать **`LINKTOROW`**:
```
LINKTOROW([_THISROW], "form_view_name")
```
Требования: целевой Form view на Slice с `Update mode: Updates only` (без Adds). Источник: pre-#08.1 §3, обнаружено при попытке вызвать форму выбытия по существующему ученику.

**BP-46. Grouped action не вызывает actions других таблиц напрямую.**
Если в один UX-шаг нужны изменения в родительской и дочерней таблицах — использовать связку: `Data: set the values` (родитель, Hide) + `Data: execute an action on a set of rows` (дочерняя таблица, Hide) + `Grouped` обёртка. Удаление строк дочерней таблицы — через вспомогательный action `Data: delete this row` на дочерней (вызывается через execute on a set of rows). Источник: pre-#08.1 §3, реализация выбытия.

**BP-47. Form Saved event — точка применения транзакции.**
Когда транзакция требует пользовательского ввода (например, ручной выбор даты), а Action with Inputs недоступен, паттерн такой: `LINKTOROW` открывает форму на Slice → пользователь сохраняет → Form Saved event запускает скрытый action или Grouped action с финальной транзакцией. Confirmation вешается на исходный action (срабатывает **до** открытия формы, что даёт UX вариант «подтверди намерение → потом введи данные»).

---

## §8. Карта подводных камней AppSheet

Накоплено в сессиях #7, #8. Полезно в брифингах для Sonnet'а.

1. **Ручной триггер Bot не существует** как отдельный тип event. Идиоматично — служебная колонка-триггер + Data Change Event.
2. **`[_THISROW_BEFORE]` / `[_THISROW_AFTER]`** работает только в Bot Event Condition, не в обычных формулах.
3. **Синтаксис `<<...>>` в Confirmation Message не работает.** Использовать `CONCATENATE`.
4. **Notification body — `CONCATENATE` работает.** Подтверждено в #07.3.
5. **«Send a notification»** шлёт push на устройство (требует AppSheet native app + системные пуши). Если push не настроены — сообщение не доставляется. **Альтернатива:** запись в settings + плитка на дашборде.
6. **`Data: execute an action on a set of rows` не работает в multi-record view (Deck/Table)** как глобальная кнопка — показывается на каждой строке. Обход: однострочная таблица + отдельный view-фрейм.
7. **`CONTEXT("View")`** возвращает имя текущего view. Использовать в Show_If.
8. **Same-table reference action** разрешён.
9. **KEY-колонку нельзя скрыть на writable-таблице.** Обход: read-only Slice.
10. **VC внутри Bot transaction:** значения пересчитываются лениво. Если в шаге `Set row values` обращаться к VC, можно получить pre-commit состояние.
14. **Snapshot целевой таблицы внутри Bot — даже на финальных шагах.** `SELECT(charges[period])` в `step_no_set_result` (после ForEach, создавшего строки) возвращает snapshot, частично или полностью НЕ учитывающий свежесозданные строки. Проявление: сообщение «Период X закрыт» с неправильным X (#16, 13.05.2026). Канон: для значений, которые нужно показать в финальных шагах бота, использовать **физическую колонку settings, обновлённую первым шагом** (вариант C+), а не пытаться вычислить через SELECT по target table. См. §3.6 и §3.11.
11. **`LINKTOFORM` ≠ `LINKTOROW`.** LINKTOFORM открывает форму создания, LINKTOROW — редактирования существующей строки. См. BP-45.
12. **VC рендерятся после физических колонок в Detail view** независимо от Column order. См. §6.9.
13. **Action with Inputs** — относительно свежая фича AppSheet, может отсутствовать в старых версиях редактора. Fallback — Slice + Form view + LINKTOROW.

Дополнительный источник — `sonnet-skills/appsheet-setup-SKILL.md` (накопленный опыт Sonnet'а по проекту, ~880 строк, Stages A–K).

---

## §9. История версий документа

- **0.1** (08.05.2026) — первый сбор. Источник: бэкапы v8, v8-fix, v9, v10, v11, v12.1 + arch-session-td1.
- **0.2** (09.05.2026) — синхронизация с отчётами pre-#08 и pre-#08.1. Состав `students` обновлён (21 колонка, +`withdrawal_date`, +`birth_date`). Добавлены §3.12 «Action chain Выбытие», §3.13 «Прочие actions UX-уровня» (`families__add_student`, `students_Form`, унификация Detail view, очистка фильтров). Добавлены хвосты §6.8–§6.11 (`_ComputedName`, VC vs Column order, Action with Inputs недоступен, предупреждение в form). Добавлен §7.3 с BP-45..BP-47 (LINKTOROW, Grouped+cross-table, Form Saved). §8 расширена пунктами 11–13.
- **0.3** (13.05.2026) — синхронизация с отчётом #15. §3.6 переписан под физическую колонку `current_period` (вместо VC). §4.3 расширен — action `settings__update_trigger` теперь пишет два значения (`last_close_trigger` + `current_period`). Зафиксирован принципиальный паттерн «VC в bot transaction = snapshot, не пересчитывается».
- **0.4** (13.05.2026, #16) — фикс варианта C+: §3.6 переписан (семантика `current_period` сдвинута — теперь «открытый период», формула в `settings__update_trigger` пишет next от max, не сам max), §3.11 переписан (формула сообщения через `ref_periods` и `student_tariffs`, не через `charges`), §4.3 и §4.5 синхронизированы. §8 расширен пунктом 14 — snapshot целевой таблицы виден и в финальных шагах бота.

---

*Этот документ описывает систему «изнутри». Бизнес-модель — в SYSTEM.md. Метафоры и термины — в GLOSSARY.md. Регламенты работы — в REGULATIONS.md.*
