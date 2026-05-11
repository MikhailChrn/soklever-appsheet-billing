# INDEX.md — Реестр артефактов проекта

Сводный индекс всех бэкапов, брифингов и отчётов. Обязательная точка входа для SubGeneral в стартовом чек-листе (после `SYSTEM.md` и `GLOSSARY.md`).

**Правило обновления:** любое создание/перенос файла в `subgeneral-backups/`, `briefings/`, `reports/` атомарно сопровождается строкой в этом индексе. См. `CLAUDE.md §5`.

---

## §1. Бэкапы SubGeneral (`subgeneral-backups/`)

| Дата | Файл | Версия AppSheet | Тип | Суть |
|---|---|---|---|---|
| 24.04.2026 | [v3](subgeneral-backups/2026-04-24_v3.md) | — | мажор | Этап 0 «Google-прародитель», старт sub-general ветки |
| 27.04.2026 | [v4](subgeneral-backups/2026-04-27_v4.md) | — | мажор | Этап 0, передача контекста |
| 28.04.2026 | [v5](subgeneral-backups/2026-04-28_v5.md) | — | мажор | SubGeneral #5, старт 28.04 |
| 29.04.2026 | [v6](subgeneral-backups/2026-04-29_v6.md) | — | мажор | SubGeneral #6, закрыт 29.04 |
| 01.05.2026 | [v7](subgeneral-backups/2026-05-01_v7.md) | — | мажор | SubGeneral #7, полное наследство |
| 02.05.2026 | [arch-session-td1](subgeneral-backups/2026-05-02_arch-session-td1.md) | 1.000337 | архитектурная сессия | Решения по ТД-1 (br05 + br05.1) |
| 03.05.2026 | [v8](subgeneral-backups/2026-05-03_v8.md) | 1.000367 | мажор | Закрытие SubGeneral #8 |
| 03.05.2026 | [v8-fix](subgeneral-backups/2026-05-03_v8-fix.md) | 1.000367 | дополнение | Дословные формулы и составы таблиц, упущенные в v8 |
| 04.05.2026 | [v9](subgeneral-backups/2026-05-04_v9.md) | 1.000367 | мажор | SubGeneral #9 (Opus) |
| 06.05.2026 | [v10](subgeneral-backups/2026-05-06_v10.md) | 1.000414+ | мажор | SubGeneral #10, после #07.3 |
| 07.05.2026 | [v11](subgeneral-backups/2026-05-07_v11.md) | 1.000427+ | мажор | SubGeneral #11, pre-#08 в работе |
| 08.05.2026 | [v12.1](subgeneral-backups/2026-05-08_v12.1.md) | 1.000440 | минор | Методологическая сессия |
| 08.05.2026 | [v12.2](subgeneral-backups/2026-05-08_v12.2.md) | 1.000440 | минор | Реструктуризация корневых файлов: создан ARCHITECTURE.md, REGULATIONS.md; обновлены SYSTEM.md, GLOSSARY.md |
| 10.05.2026 | [v12.3](subgeneral-backups/2026-05-10_v12.3.md) | 1.000507 | минор | Отчёт pre-#08.3 (закрыт частично через VC Section_Header), составлены брифы pre-#08.4 и pre-#08.5, два инцидента дисциплины SubGeneral (грep SKILL'а до брифа и при смене постановки) |
| 11.05.2026 | [v13](subgeneral-backups/2026-05-11_v13.md) | 1.000535 | мажор | Закрытие SubGeneral #13. Серия pre-#08.3..pre-#08.6 закрыта: удалены `tariff_type` и `billing_frequency`; защита «закрытый период = read-only» на Edit/Delete для charges/payments/expenses; паттерн «slice Read-Only блокирует системные actions» (SKILL R.2); плавающий дефект Delete на мобильном через slice-Detail (SKILL R.3, не блокер). Бриф #08 (летний тариф) актуализирован, готов к запуску ≤27.05. |

**Текущий рабочий бэкап:** v13 (11.05.2026).

---

## §2. Брифинги Sonnet'у (`briefings/`)

| Дата | Файл | Кому | Базовый бэкап / версия на входе | Состояние |
|---|---|---|---|---|
| 24.04.2026 | [soklever-dashboard-briefing](briefings/2026-04-24%20soklever-dashboard-briefing.md) | Ассистент по дашборду | — | Закрыт |
| 06.05.2026 | [briefing-pre08](briefings/2026-05-06_briefing-pre08.md) | Sonnet | v10, 1.000427+ | Закрыт (отчёт от 08.05) |
| 07.05.2026 | [briefing-pre08.1](briefings/2026-05-07_briefing-pre08.1.md) | Sonnet | v10 + §7.1 + pre-#08 | Закрыт (отчёт от 09.05) |
| 10.05.2026 | [briefing-pre08.3](briefings/2026-05-10_briefing-pre08.3.md) | Sonnet | v12.2 + pre-#08.1, 1.000489 | Закрыт частично 10.05 — §4.1 через компромисс VC Section_Header (раздельные заголовки Add/Edit невозможны в AppSheet); §4.2 и §4.3 закрыты. Отчёт сдан. |
| 10.05.2026 | [briefing-pre08.4](briefings/2026-05-10_briefing-pre08.4.md) | Sonnet | pre-#08.3, 1.000507 | Закрыт 11.05.2026 — `tariff_type` удалён полностью (фильтр в `bot_close_period`, дополнительно правка `settings[current_period]`, колонка в Sheets). Отчёт сдан. |
| 10.05.2026 | [briefing-pre08.5](briefings/2026-05-10_briefing-pre08.5.md) | Sonnet | pre-#08.4, 1.000518 | Закрыт 11.05.2026 — 5 задач (правки 11.05: §4.4 переписан под `[period] = current`, добавлена §4.5 по `charge_type`). Все 5 закрыты. |
| 11.05.2026 | [briefing-pre08.6](briefings/2026-05-11_briefing-pre08.6.md) | Sonnet | pre-#08.5, 1.000523 | Закрыт 11.05.2026 — обе задачи брифа + плановое расширение в ходе сессии (Edit/Delete на charges/payments/expenses, slice-режимы, ref_periods Read-Only). |
| 11.05.2026 | [briefing-08](briefings/2026-05-11_briefing-08.md) | Sonnet | v12.3 + pre-#08.3..pre-#08.6, 1.000535 | Открыт. Массовая замена тарифа на летний, дедлайн ≤27.05. Исходный черновик 09.05.2026, актуализирован 11.05.2026: учтены удаления колонок `tariff_type` (pre-#08.4) и `billing_frequency` (pre-#08.6); добавлено замечание о том, что защиты pre-#08.5/08.6 не влияют на скрипт (он не трогает charges/payments/expenses). |

---

## §3. Отчёты Sonnet'а (`reports/`)

| Дата | Файл | По брифингу | Версия вход → выход | Итог |
|---|---|---|---|---|
| 06–08.05.2026 | [report-pre08](reports/2026-05-08_soklever-appsheet-report-pre08.md) | pre-#08 | 1.000429 → 1.000441 | Сдан |
| 09.05.2026 | [report-pre08_1](reports/2026-05-09_soklever-appsheet-report-pre08_1.md) | pre-#08.1 | 1.000444 → 1.000483 | Сдан, все 5 задач закрыты; 3 архитектурных хвоста для стратега |
| 10.05.2026 | [report-pre08_3](reports/2026-05-10_soklever-appsheet-report-pre08_3.md) | pre-#08.3 | 1.000489 → 1.000507 | Сдан частично. §4.1 закрыт компромиссом (VC Section_Header «Создать/редактировать …») — раздельные заголовки Add/Edit невозможны в AppSheet. §4.2 и §4.3 закрыты. Финальный бэкап post-08-3 не сделан Sonnet'ом — выполнить владельцу. |
| 11.05.2026 | [report-pre08_4](reports/2026-05-11_soklever-appsheet-report-pre08_4.md) | pre-#08.4 | 1.000507 → не зафиксирована | Сдан полностью. `tariff_type` удалён: правка фильтра в `bot_close_period`, плюс обнаружена и исправлена дополнительная ссылка в `settings[current_period]` (всплыла при Regenerate Schema). Финальный бэкап не сделан — тестовая схема, решение владельца. Хвост: внебрифинговое создание/удаление view `начисления` + ресёрч встроенного фильтра (правило формирования списка фильтруемых колонок не понято). |
| 11.05.2026 | [report-pre08_5](reports/2026-05-11_soklever-appsheet-report-pre08_5.md) | pre-#08.5 | 1.000518 → 1.000523 | Сдан полностью. §4.1 `billing_frequency` — скрыта в AppSheet (Show? = false), удаление из Sheets отложено в pre-#08.6. §4.2 форма ученика — `students_visible_Form` очищена от VC и Related. §4.3 inline платежи — закрыта по факту (карточка ученика отображает тип/сумму/дату/комментарий корректно). §4.4 Edit charges — Show_If = `[period] = INDEX(settings[current_period], 1)`. §4.5 `charge_type` — Initial value «Ручное» уже стояло, поле убрано из формы; бот через `student_tariffs__create_next_charge` явно пишет «Авто». Финальный бэкап не сделан (тестовая схема). Хвосты: удаление `billing_frequency` из Sheets, защита от создания charges в закрытом периоде. |
| 11.05.2026 | [report-pre08_6](reports/2026-05-11_soklever-appsheet-report-pre08_6.md) | pre-#08.6 | 1.000523 → 1.000535 | Сдан полностью. §4.1 `billing_frequency` удалён из Sheets, Regenerate Schema без ошибок. §4.2 Valid_if на `charges[period]` (текущий + 3 вперёд) + Initial value; `ref_periods` → Read-Only (закрыли «+ New»). Плановое расширение в ходе сессии: Edit + Delete защищены от закрытого периода на charges (`[period] = current`), payments (`TEXT([payment_date], "YYYYMM") = current`), expenses (`[period] = current`); slice'ы `charges_filtered`, `payments_filtered`, `expenses_filtered` переведены из Read-Only в Updates+Deletes (паттерн «slice Read-Only блокирует системные actions»); исправлен баг `expenses[period]` Valid_if (была `- 3` на нижней границе). Финальный бэкап не сделан (тестовая схема). Хвосты: Delete не виден на мобильном через slice-Detail (charges, expenses; payments — виден); ревизия всех slice-ов проекта на Read-Only; ревизия всех Valid_if на ref_periods на наличие `- 3`. |

---

## §4. Что не вошло, но смежно

- `sonnet-skills/` — рабочие SKILL'ы Sonnet'а (актуальная версия каждого, не история). Не индексируется здесь, потому что это не append-only артефакты.
- `archive/` — устаревшее, в индексе не отражается.
- `reference-data/` — задел на брифинг #09 (Disaster Recovery). На 11.05.2026 содержит:
  - `2026-04-16 soklever-formulas-guide-v7.md` — инструкция по формулам Google Sheets (эпоха до AppSheet).
  - `2026-04-17 soklever-google-ancestor-v6.xlsx` — эталонная схема книги.
  - `2026-05-09 soklever-google-ancestor-v6 (main).xlsx` — текущий рабочий снимок.

---

## §5. История изменений индекса

| Дата | Что |
|---|---|
| 09.05.2026 | Файл создан в SubGeneral #13. Заполнен по факту: 13 бэкапов, 3 брифинга, 2 отчёта. |
| 09.05.2026 | §4 расширен описанием `reference-data/` (задел на #09). §2 — добавлен `briefing-08` (массовая замена тарифа). |
| 10.05.2026 | §2 — добавлен `briefing-pre08.3` (UX-патч заголовков форм). |
| 10.05.2026 | §3 — добавлен `report-pre08_3` (закрыт частично, компромисс VC Section_Header). §2 — статус `briefing-pre08.3` обновлён. |
| 10.05.2026 | §2 — добавлены `briefing-pre08.4` (чистка рудиментов: tariff_type, view-фильтры) и `briefing-pre08.5` (UX-чистка). |
| 10.05.2026 | §1 — добавлен `v12.3`. Текущий рабочий бэкап обновлён. |
| 11.05.2026 | §2 — статус `briefing-pre08.4` обновлён (закрыт). §3 — добавлен `report-pre08_4`. |
| 11.05.2026 | §2 — статус `briefing-pre08.5` обновлён (закрыт). §3 — добавлен `report-pre08_5`. |
| 11.05.2026 | §2 — добавлен `briefing-pre08.6` (удаление billing_frequency из Sheets + Valid_if на charges[period]). |
| 11.05.2026 | §2 — статус `briefing-pre08.6` обновлён (закрыт). §3 — добавлен `report-pre08_6`. |
| 11.05.2026 | §2 — `briefing-08` актуализирован и переименован в `2026-05-11_briefing-08.md` (синхронизирован с pre-#08.3..pre-#08.6: удаление `tariff_type`/`billing_frequency`, версия 1.000535). |
| 11.05.2026 | §1 — добавлен `v13` (мажор, закрытие SubGeneral #13). Текущий рабочий бэкап обновлён. |
