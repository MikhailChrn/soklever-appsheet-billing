---
name: appsheet-setup
version: 2.19
description: >
  Step-by-step guide for setting up AppSheet applications connected to Google Sheets.
  Use this skill whenever a user wants to: create an AppSheet app from scratch, configure
  table columns and types, set up virtual columns with formulas, configure views and
  navigation, set up security filters and roles, brand the app, build a dashboard,
  or troubleshoot AppSheet issues. Also use when a user asks about Ref columns, Enum types,
  security filters, Format Rules, AppSheet expressions, Slices, or Dashboard views.
  Also use when a user asks about App Documentation export, Enum audit, or managing
  period/class Enum synchronisation across multiple tables.
  Also use when a user asks about Enum-to-Ref refactoring, ref_periods/ref_classes
  reference tables, Valid_If on Ref columns, or auto-generated Ref views/actions cleanup.
  Also use when a user asks about Automation, Bots, Bot events, manual Bot triggers,
  action sequences, Branch on condition, or Run a data action in Bots.
  Also use when a user asks about Bot step type change, Bot step renaming, anti-cascade
  Branch condition pattern, or KEY column visibility in detail views.
  Especially useful for developers new to AppSheet who understand relational databases
  but need AppSheet UI guidance.
---

# AppSheet Setup Guide v2.19

This skill captures the end-to-end setup of a production AppSheet app connected to Google Sheets.

## User Profile

Target user: developer who understands relational databases, foreign keys, and enums — but is new to AppSheet. Skip DB theory, focus on WHERE things are in the AppSheet Editor UI.

## Setup Order (Stages A–K)

### A. Prepare Google Sheets

1. In the `expenses` sheet, add a column `created_by` (lowercase, no spaces). Leave empty — AppSheet fills it automatically via `USEREMAIL()`.
2. Create a new sheet `users` with columns `email` and `role`. Add ADMIN rows. OPERATOR rows can be added later when a real employee joins.

### B. Create the App

1. Go to appsheet.com → **+ Create** → **App** → **Start with existing data**
2. Select **Google Sheets**, choose your file
3. Name the app, choose category → **Choose your data**
4. AppSheet will connect the first sheet automatically. After opening the editor, click **+** next to **Data** to add remaining sheets.
5. In the sheet selector dialog: **uncheck** any sheets that should NOT be connected (e.g., `debts`, `dashboard` — computed elsewhere).
6. Click **Add N tables**.

### C. Configure Column Types

Navigate to **Data** → click each table → adjust types.

**Key rules:**
- Foreign key columns → type **Ref**, set Source table
- Phone fields → type **Phone**
- Date-only fields → type **Date** (not DateTime)
- Auto-generated IDs → keep as **Text**
- `created_by` → App formula = `USEREMAIL()`; uncheck Show?
- `created_at` → type **DateTime**, Initial value = `NOW()`; hide from forms

**Enum columns:** For period fields (e.g., `202604`), change type to Enum and enter allowed values manually.

**After saving**, AppSheet shows warnings about hidden columns being unsearchable — informational only.

### D. Virtual Columns (Computed Fields)

Virtual columns are calculated fields stored only in AppSheet, not in Sheets.

**To add:** Data → table → click **+** button (top right of column list).

**Debt calculation for student:**
```
SUM(SELECT(charges[amount], [student_id] = [_THISROW].[student_id]))
- SUM(SELECT(payments[amount], [student_id] = [_THISROW].[student_id]))
- SUM(SELECT(opening_balances[amount], [student_id] = [_THISROW].[student_id]))
```

**Family total debt:**
```
SUM(SELECT(students[debt], [family_id] = [_THISROW].[family_id]))
```

**Important:** Use `[_THISROW].[column]` to reference the current row in SELECT. Without it, the filter compares column to itself and returns all rows.

**Testing:** Use the **Test** button in Expression Assistant. Note: virtual columns cannot reference other virtual columns from the same table in Test context — they return 0. Works correctly in live app.

### E. Views

Go to **Views** (phone icon in left panel).

**To add views:** click **+** next to PRIMARY NAVIGATION → **Create a new view**.

**Recommended view types:**
- Lists of records → **deck**
- Reference/lookup tables → **table**
- Dashboard → **dashboard**

**Key deck settings:**
- **Primary header** → display name column
- **Secondary header** → descriptive field
- **Summary column** → numeric value

**Sort:** Add sort in View Options. Use Descending for date/debt fields.

**Show if:**
```
LOOKUP(USEREMAIL(), "users", "email", "role") = "ADMIN"
```

**Dynamic card header (Detail view):**
Add column to **Header columns** in View Options — it renders as large bold title at top of the card. Use `_ComputedName` virtual column for dynamic titles like `CONCATENATE([first_name], " ", [last_name], " (", [class_group], ")")`.

### F. Forms

AppSheet auto-generates forms. To customize:
1. Find form in **SYSTEM GENERATED** section
2. Switch **Column order** to **Manual**
3. Hide: auto-generated IDs, `created_at`, virtual columns, `Related X` lists

**Display names:** Data → table → column → pencil → Display section → **Display name**.

**⚠️ Form Column order Manual:** When switching to Manual and adding columns, add ONLY columns from that table. AppSheet may inject columns from other tables (e.g., `settings`) into the list — these must not be included.

**Form title / section header:** AppSheet does not have a dedicated form title field. Use a virtual column of type `Show` with category `Section_Header` as the first column in the form. See **Q.16** for the full pattern.

**⚠️ KEY column cannot be hidden via Show?** on a writable table. To hide a KEY column from a form, use Column order Manual and simply do not add it to the list. Alternatively, set the table to Updates-only first.

### G. Actions

AppSheet auto-creates Delete and Edit actions. Skip custom actions for initial setup.

**⚠️ Action "App: go to another view within this app"** — does NOT work reliably for ref-position views (Network error, see §Q.5), neither in preview nor on mobile. Use direct navigation via PRIMARY NAVIGATION instead.

**Action "App: open a form to add a new row"** — works correctly. Attach to the table that owns the form view (e.g., `expenses` for adding expenses). Position: Prominent shows as button, Inline shows in action bar.

**Action "Data: add a new row to another table using values from this row"** — creates the row immediately, without opening a form. The user cannot fill in additional fields at click time. Use only when ALL required fields can be derived from the source row's context. For "create child from parent context where user must enter name/class/etc." — use LINKTOFORM pattern below instead.

**LINKTOFORM for pre-filled forms:**
```
LINKTOFORM("FormViewName", "column1", [value1], "column2", [value2])
```
⚠️ All columns referenced in LINKTOFORM must have Show? = true. If a column is hidden (Show? = false), AppSheet throws an error and the action fails. KEY columns used in LINKTOFORM cannot be hidden.

**Pattern — open a pre-filled child form from parent's Detail view** (e.g., "+ child" button on family card):

Action on parent table, type = "App: go to another view within this app", Target =
```
LINKTOFORM("Child_Form_View", "foreign_key_col", [parent_key])
```

This is the **exception** to the "go to another view" warning above: that action type fails for raw view navigation to ref-position views, but **works correctly** when Target is a LINKTOFORM(...) formula — LINKTOFORM packages a form-open intent, not a view URL. Opens the child form with the foreign key pre-filled; the user fills in remaining fields and Saves.

⚠️ Do NOT try to gate visibility of fields in the child form via `Show_If = ISNOTBLANK([key])` — on a create form the KEY is pre-generated by AppSheet, so the condition is always true and the gate is a no-op. To hide system/internal columns on the create form, use **Column order = Manual** in the system-generated form view and omit those columns from the order.

**LINKTOFORM is create-only — it cannot open an existing row for editing.** Attempts to "edit via LINKTOFORM" by passing the existing key fail with `There is already a row with the key: '<KEY>'`. For editing an existing row in a dedicated form (e.g., to capture a single field like `withdrawal_date` before applying a transaction), use **LINKTOROW** instead:

```
LINKTOROW([_THISROW], "Edit_Form_View_Name")
```

**Pattern — open an existing row in a narrow Edit form** (e.g., "ask user to pick a date, then apply withdrawal transaction"):

1. Add a service column on the table to hold the user input (e.g., `students[withdrawal_date]`, type Date, Show? = FALSE in default views).
2. Create a Slice over that table, **Updates only**, columns = the service column + KEY.
3. Create a Form view on that Slice (`Edit_Form_View_Name` above) with Column order containing only the service column.
4. Action on the source row, type = `App: go to another view within this app`, Target = `LINKTOROW([_THISROW], "Edit_Form_View_Name")`. This opens the form in **Edit mode** on the existing row — no "duplicate key" error.
5. On the Form Saved event of that form view, attach a Bot/Action that reads the service column and applies the real transaction (set the destination columns, run cross-table actions, etc.).

Differences:
- `LINKTOFORM(view, col, val, …)` — creates a new row in `view`, pre-filling specified columns.
- `LINKTOROW(rowref, view)` — opens an existing row (referenced by `rowref`, typically `[_THISROW]`) in `view` — works for both Detail and Form views; on a Form view it acts as Edit.

**Hiding Delete action:** UX → Actions → table → Delete → Position = `Hide`. Prevents accidental deletion without removing the action entirely.

### H. Security

**Step 1 — Add users:** Security → Require sign-in → **Manage users**.

**Step 2 — Security Filters:**
For ADMIN-only tables:
```
LOOKUP(USEREMAIL(), "users", "email", "role") = "ADMIN"
```

For OPERATOR own-records:
```
OR(
  LOOKUP(USEREMAIL(), "users", "email", "role") = "ADMIN",
  [created_by] = USEREMAIL()
)
```

**Step 3 — View visibility:** Hide from OPERATOR using **Show if** in Display section.

### I. Branding

Settings (gear icon) → **Theme & Brand** → Primary color hex.

### J. Testing Checklist

1. Formula verification against known correct values
2. ADMIN role: all views visible, full CRUD
3. OPERATOR role: only allowed views, only own records
4. Form usability: required fields, auto-generated IDs, dropdowns
5. Debt recalculation after adding payment

### K. Dashboard (Admin Summary View)

#### K.1 Architecture: Settings Table Pattern

AppSheet Dashboard view has a critical limitation: **no row limit on Deck views inside Dashboard**. For production apps with large datasets, inline Deck blocks show all records — unusable.

**Recommended pattern:**
1. Create a Google Sheet `settings` with one row: `id=1, label=dashboard`
2. Connect to AppSheet as Updates-only table (enables filter field editing)
3. Add virtual columns to `settings` — each computes one dashboard metric
4. Create a Detail view on `settings`
5. Add Detail view as entry in Dashboard view

#### K.2 Table Settings Setup

**In Google Sheets:** sheet `settings` with columns `id` (value: 1), `label` (value: dashboard). Add real filter columns as needed: `filter_category`, `filter_period`.

**In AppSheet:**
- Data → **+** → select `settings` → Add table
- Table settings (gear ⚙️) → set **Updates only** (disables Adds/Deletes, allows editing filter fields)
- To add new columns from Sheets: click 🔄 (regenerate) next to gear → **Regenerate**

**⚠️ After Regenerate:** new columns may get wrong types (e.g., `filter_period` detected as Duration instead of Text). Always verify and fix types manually after regeneration.

**⚠️ Date column auto-formula after Regenerate:** When a Date column (e.g., `academic_year_start`) is added to Sheets and then pulled via Regenerate Schema, AppSheet may auto-assign an App formula like `IF(MONTH(TODAY())...)`. This overwrites the Sheets value. After Regenerate, always open the column → Auto Compute → verify App formula is **empty**. Clear it if not.

#### K.3 Key Virtual Column Formulas

**Current period** (last period with regular charges):
```
INDEX(SORT(SELECT(charges[period], [tariff_id].[tariff_type] = "Регулярный"), TRUE), 1)
```
⚠️ `MAX()` does NOT work on Enum/Text columns. Use `INDEX(SORT(..., TRUE), 1)` instead.

**Monthly receipts** (period is Enum/Text — use string comparison):
```
SUM(SELECT(payments[amount], [period] = [current_period]))
```
⚠️ Comparing Enum column to number literal causes type error. Always use string or reference another Enum/Text column.

**Monthly expenses:**
```
SUM(SELECT(expenses[amount], [period] = [current_period]))
```

**Debtor count:**
```
COUNT(SELECT(families[family_id], [family_debt] > 0))
```

**Total debt of debtors:**
```
SUM(SELECT(families[family_debt], [family_debt] > 0))
```

**Academic year start** — store as a fixed Date value in Google Sheets `settings` table, column `academic_year_start`. Do NOT compute dynamically — the formula requires knowing whether the current school year has started, which may conflict with app logic.

**Open period receipts** (payments in current_period month/year):
```
SUM(
  SELECT(
    payments[amount],
    AND(
      YEAR([payment_date]) = NUMBER(LEFT(ANY(settings[current_period]), 4)),
      MONTH([payment_date]) = NUMBER(RIGHT(ANY(settings[current_period]), 2))
    )
  )
)
```

**Open period expenses** (by period field):
```
SUM(SELECT(expenses[amount], NUMBER([period]) = NUMBER(ANY(settings[current_period]))))
```

**Closed periods receipts** (from academic_year_start up to but not including current_period):
```
SUM(
  SELECT(
    payments[amount],
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
  )
)
```

**Closed periods expenses** (by period field, >= academic year start, < current_period):
```
SUM(
  SELECT(
    expenses[amount],
    AND(
      NUMBER([period]) >= (YEAR(ANY(settings[academic_year_start])) * 100 + MONTH(ANY(settings[academic_year_start]))),
      NUMBER([period]) < NUMBER(ANY(settings[current_period]))
    )
  )
)
```

**Previous period** (YYYYMM, handles December→January boundary):
```
IFS(
  NUMBER(RIGHT(ANY(settings[current_period]), 2)) > 1,
  NUMBER(LEFT(ANY(settings[current_period]), 4)) * 100 + NUMBER(RIGHT(ANY(settings[current_period]), 2)) - 1,
  TRUE,
  (NUMBER(LEFT(ANY(settings[current_period]), 4)) - 1) * 100 + 12
)
```
⚠️ Simple `NUMBER(current_period) - 1` breaks on January (202601 - 1 = 202600). Always use this IFS pattern.

**YTD receipts:**
```
SUM(SELECT(payments[amount], [payment_date] >= [_THISROW].[academic_year_start]))
```

**YTD expenses:**
```
SUM(SELECT(expenses[amount], [expense_date] >= [_THISROW].[academic_year_start]))
```

**Free funds (YTD):**
```
[поступления_год] - [расходы_год]
```

#### K.4 Dashboard View Setup

**Detail view on settings:**
- View name: `дашборд_цифры`
- For this data: `settings` (or a slice of settings)
- View type: `detail`
- Position: `ref`
- **Display name formula (dynamic title — KEY PATTERN):**
  ```
  CONCATENATE("Текущий период: ", INDEX(settings[current_period], 1))
  ```
  ⚠️ View Display name has no row context — use `ANY(settings[column])` or `INDEX(settings[column], 1)`, NOT `[column]`.
  
  **This is the correct way to make dynamic block headers in Dashboard.** The Display name of a ref-position Detail view becomes the block header shown in Dashboard. Use a formula with `ANY()` or `INDEX()` to make it dynamic.

- Hide service columns: `id`, `label`, `current_period`, `academic_year_start` → Show? = unchecked
- Column order: Manual — drag to desired order
- **Column Display names:** Data → settings → column → pencil → Display section → **Display name**. Set human-readable names here (not in the view). This affects all views that show the column.

**Dashboard view:**
- View type: `dashboard`
- Position: `first`
- Show if: `LOOKUP(USEREMAIL(), "users", "email", "role") = "ADMIN"`
- View entries: add detail views (size: Large)

**⚠️ When Dashboard has multiple sub-views**, AppSheet shows a block header above each sub-view. This header is taken from the **Display name** of the sub-view (not its View name). Set Display name to a formula to make it dynamic and human-readable.

**⚠️ When Dashboard has only one sub-view**, the block header is NOT shown — only the content of the sub-view is visible. This is why the old single-block dashboard looked clean.

#### K.5 Multi-block Dashboard Pattern

For dashboards with multiple thematic blocks (e.g., open period / debts / closed periods):

1. Create one slice per block on `settings` — each slice contains only the columns for that block.
2. Create one Detail view per slice (Position: ref). Set Display name = formula.
3. Set Column order = Manual in each Detail view, add only the relevant columns (exclude `id`).
4. Create Dashboard view, add all Detail views as View entries.

**⚠️ KEY column (`id`) in slice:** cannot be excluded from Slice Columns. It will always appear. To hide it from the Detail view — use Column order Manual and do NOT add `id` to the list. It will not be shown even though it exists in the slice.

**Naming convention for dashboard slices and views:**
- Slices: `settings_dash_[block]` (e.g., `settings_dash_открытый`)
- Detail views: `dash_[block]_detail` (e.g., `dash_открытый_detail`)

#### K.6 Slices

**⚠️ Where to CREATE a new Slice:**
- Data → hover over table name → click **+** that appears → **New Slice**
- This pre-sets the source table correctly.

**⚠️ Where to FIND existing Slices:**
- **Data** → expand a table → click the slice listed under it
- Or via the view that uses it: Views → view → "For this data" dropdown

**To delete a Slice:** Data → expand table → click slice → three dots "..." top right → Delete.

**⚠️ Slice formula not saving bug:** If a Slice has an invalid formula, Expression Assistant may show green checkmark but not save the new formula. Solution: delete the Slice and recreate it from scratch.

**⚠️ Slice deletes new VC on save:** If a Slice has a restricted Slice Columns list, it will delete any new Virtual Column that is not already in that list upon saving. Solution: create the VC first, then immediately add it to the Slice Columns list BEFORE saving. Do both in one session without saving in between.

**Slice for debtors only:**
- Slice name: `должники_slice`
- Source table: `families`
- Row filter: `[family_debt] > 0`
- Update mode: `READ_ONLY` (removes "+" button from the view)

**Slice for filtered expenses (by period and category, with ADMIN/OPERATOR role filter):**
```
AND(
  OR(
    ISBLANK(ANY(settings[filter_period])),
    [period] = ANY(settings[filter_period])
  ),
  OR(
    ISBLANK(ANY(settings[filter_category])),
    TEXT([category]) = TEXT(ANY(settings[filter_category]))
  ),
  OR(
    LOOKUP(USEREMAIL(), "users", "email", "role") = "ADMIN",
    [created_by] = USEREMAIL()
  )
)
```
⚠️ The third OR condition implements role-based filtering: ADMIN sees all, OPERATOR sees only their own records. This pattern applies to any slice where ownership filtering is needed.

⚠️ When `category` is a Ref column and `filter_category` is also Ref, direct comparison `[category] = ANY(settings[filter_category])` may fail. Use `TEXT()` wrapper on both sides.

⚠️ `ANY(settings[column])` reads from the single-row settings table. Works correctly only when settings has exactly one row.

**Slice for recent payments (last N days):**
```
[payment_date] >= (TODAY() - 150)
```

**Slice for recent charges by period (last N days, YYYYMM Enum):**
```
NUMBER([period]) >= (YEAR(TODAY() - 150) * 100 + MONTH(TODAY() - 150))
```
See Section M for explanation of the NUMBER() pattern.

#### K.7 Filter Panel Pattern (расходы с фильтром)

Use a single-row `settings` table as the filter state store:
1. Add filter columns to `settings` sheet: `filter_period`, `filter_category`, etc.
2. In AppSheet, set their type to match the filtered column (Enum or Ref).
3. Slice row filter uses `OR(ISBLANK(ANY(settings[filter])), [col] = ANY(settings[filter]))`.
4. Create a Detail view on `settings` (Position: ref) as the filter panel UI.
5. Show the filter panel via a navigation action or menu entry.

---

## Section P — Enum-to-Ref Refactoring

This section documents the full procedure for converting Enum columns to Ref columns backed by reference tables (`ref_periods`, `ref_classes`, etc.).

### P.1 Why Refactor

- **Single source of truth:** add a new period/class once in the sheet, all tables see it
- **No synchronisation errors:** eliminates the recurring problem of Enum values being out of sync across tables
- **Scalable:** `ref_periods` can grow to cover any date range without touching AppSheet config

### P.2 Reference Table Setup in Google Sheets

**`ref_periods` sheet:**
- Column A header: `period`
- Values: YYYYMM as Text (e.g., `202601`, `202602`, ..., `203612`)
- Recommended range: current year - 1 to current year + 5 (72–120 rows)

**Fast fill method (formula helper):**
1. In B1 enter a start date: `2026-01-01`
2. In B2: `=B1+32` — extend to B121 (each row guaranteed to be a different month)
3. In A2: `=YEAR(B2)*100+MONTH(B2)` — extend to A121
4. Select A2:A121 → Copy → Paste special → Values only
5. Delete column B

⚠️ The +32 trick works because adding 32 days to any date always lands in the next month. Sheets handles date math correctly across month/year boundaries.

**`ref_classes` sheet:**
- Column A header: `class_group`
- Values: `0 класс`, `1 класс`, ..., `11 класс` (12 rows)
- Values must match exactly what is stored in the `students` table — including spaces and case

### P.3 Adding Reference Tables to AppSheet

1. Data → **+** → Google Sheets → select same file → select `ref_periods` sheet
2. Verify: `period` column → type **Text**, KEY? = ✅
3. Repeat for `ref_classes`: `class_group` → type **Text**, KEY? = ✅
4. Save

**⚠️ Type must be Text, not Number.** If AppSheet detects the column as Number, the period values will display with thousands separator (e.g., `202 603` instead of `202603`). Change type to Text.

**⚠️ AppSheet auto-generates views for new reference tables** (`ref_periods_Detail`, `ref_periods_Form`, `ref_classes_Detail`, `ref_classes_Form`). These are not needed if users manage the reference data directly in Google Sheets. Delete all auto-generated views for reference tables:
UX → Views → SYSTEM GENERATED → ref_periods (2) → delete both → repeat for ref_classes.

**⚠️ AppSheet auto-generates `View Ref (period)` actions** for every table that has a Ref column pointing to `ref_periods`. These actions open the reference table card — not useful for end users. Hide or delete them:
Behavior → Actions → expand each table → find `View Ref (period)` / `View Ref (class_group)` → Position = Hide (or Delete if allowed).

### P.4 Converting Enum Columns to Ref

For each column being converted (e.g., `charges[period]`, `expenses[period]`, `students[class_group]`, `settings[filter_charges_period]`, etc.):

1. Data → table → column → pencil ✏️
2. Type → change to **Ref**
3. Source table → select `ref_periods` (or `ref_classes`)
4. Is a part of? → **false** (reference tables are not owned by the parent record)
5. Done → Save

**Order matters:** Convert columns one at a time. Save and verify the app loads after each conversion for the first few — AppSheet handles Ref cascades gracefully but it's safer to proceed incrementally.

**⚠️ Before deleting any column** (e.g., a redundant computed column like `period_num`): audit App Documentation for all references. Download HTML documentation via:
`https://www.appsheet.com/template/AppDoc?appId=YOUR_APP_ID`
Then search for the column name. Check: Slice Columns lists, Virtual Column formulas, View Column orders, Action formulas. Remove from all locations before deleting from Sheets.

### P.5 Valid_If on Ref Columns (Dropdown Filtering)

**Common misconception:** Valid_If on a Ref column validates after selection. **This is wrong.**

Valid_If on a Ref column **filters the dropdown** if the expression returns a **list of key values** from the referenced table. If Valid_If returns Yes/No, it only validates — it does NOT filter the dropdown.

**Correct pattern — return a filtered list of keys:**
```
SELECT(
  ref_periods[period],
  AND(
    NUMBER([period]) >= NUMBER(ANY(settings[current_period])) - 3,
    NUMBER([period]) <= NUMBER(ANY(settings[current_period])) + 3
  )
)
```

**Wrong pattern — returns Yes/No, does NOT filter dropdown:**
```
AND(
  [_THIS] >= ANY(settings[current_period]) - 3,
  [_THIS] <= ANY(settings[current_period]) + 3
)
```

**⚠️ Arithmetic on Text keys requires NUMBER() wrapper.** After converting period from Enum to Text Ref, `ANY(settings[current_period])` returns Text. Arithmetic on Text causes `inputs of an invalid type 'Unknown'` error. Wrap both sides in `NUMBER()`.

**⚠️ VALUE() and TEXT() are not AppSheet functions for type conversion.** Use `NUMBER()` to convert Text to numeric. `VALUE()` does not exist in AppSheet expressions.

**For class dropdown (show all classes, no filter):**
```
SELECT(ref_classes[class_group], TRUE)
```

**Business anchor for period window:** Use `current_period` from `settings` (last period with regular charges), NOT `TODAY()`. This ensures the dropdown window follows business activity, not the calendar. If charges are entered for April but the calendar shows August, the window should be around April.

**Window size:** ±3 months covers most retroactive entry scenarios for a school. On year boundaries, the SELECT formula naturally clips to available values in `ref_periods` — no special handling needed.

### P.6 Initial Value on Ref Period Columns

After setting Valid_If, add Initial value so the period auto-fills in forms:
```
ANY(settings[current_period])
```

Apply to: `charges[period]`, `expenses[period]` (and any other entry forms where period should default to current).

### P.7 Phantom Column Cleanup After Refactor

After an Enum-to-Ref refactor, scan for phantom references to the old column name:

**In Views (Column order):**
- Forms: find old column name in Manual column order → remove
- Deck/Table views: check Secondary header, Summary column for old column

**In Actions:**
- Auto-generated `View Ref (X)` actions for new Ref columns → Hide/Delete

**Procedure:**
1. Download App Documentation HTML
2. `grep -n "old_column_name" documentation.html` (or Ctrl+F in browser)
3. For each hit: remove from Slice Columns, View Column orders, Action formulas
4. Then delete from Google Sheets + Regenerate in AppSheet

---

## Troubleshooting

**New column from Sheets gets wrong type after Regenerate**
Always verify column types manually after Regenerate. Common: period/text fields detected as Duration or Price.

**Date column gets auto-formula after Regenerate**
AppSheet may assign `IF(MONTH(TODAY())...)` to a Date column pulled from Sheets. After Regenerate, open the column → Auto Compute → clear App formula if present.

**Action "navigate to view" returns Network error**
Known AppSheet limitation — navigation Actions to ref-position views don't work. Use menu navigation instead.

**Detail view on settings shows all columns (including dashboard virtuals)**
Switch Column order to Manual and add only needed columns via + Add. Never use Automatic on settings detail views used as filter panels.

**Form Column order Manual injects foreign columns**
When switching form to Manual column order, AppSheet may show columns from other tables. Add only the columns belonging to that form's table.

**Section header not appearing in form**
1. Check column is added to form's Column order (Manual → + Add).
2. Check Show? = true (no conditions).
3. KEY column with UNIQUEID() Initial value is never blank — do not use ISBLANK([key]) as Show? condition.

**Deck view renders as table instead of deck**
Three possible causes (check all):
1. Primary header / Secondary header / Summary column are empty → fill them explicitly.
2. No Label column on source table → set Label = ✅ on one column (recommended: VC `_ComputedKey`).
3. `Related X` column points to the full table, not a slice → change Referenced table name to the slice.

**Related X shows technical table in Detail card instead of deck**
Change Referenced table name on the `Related X` column from the full table to a slice:
Data → parent table → `Related X` → Type Details → Referenced table name → select `slice_name (slice)`.
A slice is required even if it has no row filter — AppSheet uses the slice's presence to trigger deck rendering.

**Slice deletes new VC on save**
Slice has restricted Slice Columns list. Create VC → immediately add to Slice Columns → then Save. Never save between creating VC and adding it to slice.

**Format Rules not found**
Format Rules are NOT under the gear ⚙️ next to "Views". Correct path:
UX (phone icon) → second column submenu → **Format Rules**.

**Dashboard block header shows technical view name**
The block header in a multi-entry Dashboard is taken from the **Display name** of the sub-view. Set Display name to a formula in the view's Display section (Go to display options → Display name). Use `ANY(settings[column])` since there is no row context.

**Previous period formula gives 202600 for January**
Simple `NUMBER(current_period) - 1` breaks on year boundary. Use the IFS pattern (see Section M, Pattern 5).

**OPERATOR sees all records instead of only their own**
Add role-based OR condition to slice Row filter:
```
OR(
  LOOKUP(USEREMAIL(), "users", "email", "role") = "ADMIN",
  [created_by] = USEREMAIL()
)
```
If still not working — check email case sensitivity. Wrap in `LOWER()`:
```
OR(
  LOOKUP(USEREMAIL(), "users", "email", "role") = "ADMIN",
  LOWER([created_by]) = LOWER(USEREMAIL())
)
```

**Колонка есть в форме, но отсутствует в Documentation**
Если колонка видна в Column order формы (Manual), но не отображается в HTML Documentation — она скрыта (`Show? = false`). Documentation не экспортирует скрытые колонки. Для полного списка колонок таблицы всегда проверяй редактор, а не только Documentation.

**App Documentation не найден в Manage**
В текущей версии редактора AppSheet раздел Documentation/Author отсутствует в меню Manage (видны только Deploy / Versions / Monitor / Collaborate & Publish). Использовать прямую ссылку: `https://www.appsheet.com/template/AppDoc?appId=YOUR_APP_ID`. App ID берётся из адресной строки редактора. Подробнее — Section N.

**Enum рассинхронизирован в одном из мест, фильтр не работает**
Одна из причин — опечатка в EnumValues (например, `"11класс"` вместо `"11 класс"`). Проверить все Enum-определения по карте Section O.2. Вторая причина — новый период/класс добавлен не во все колонки (см. чеклисты O.5 и O.6).

**Ref-колонка не появляется в списке добавления колонок формы**
Если Ref-колонка не отображается в списке Column order (Manual → + Add) формы — причина в Show? = false на этой колонке. Открой Data → таблица → колонка → Show? = true. После этого колонка появится в списке формы.

**VC типа Show не вычисляется в форме для новой строки**
VC типа Show с формулой на основе Ref (например, CONCATENATE([ref_id].[field], ...)) не отображает значение в форме добавления новой записи — контекст строки ещё не существует. Решение: поменяй тип VC на Text. Тип Text с App formula вычисляется в форме корректно и показывает значение из Ref-колонки контекста.

**Referenced table name принимает только таблицы и слайсы, не views**
При настройке Related X колонки (Type Details → Referenced table name) в списке отображаются только таблицы и слайсы. Отдельный ref-position view в этом списке не появится. Для кастомного отображения Related-блока создай slice на нужной таблице и укажи его в Referenced table name.

**Конфликт лишнего Table view и inline Deck — Related-блок отображается как таблица**
Симптом: в карточке родительской записи Related-блок внезапно показывается как таблица с неправильными колонками вместо настроенного deck.
Причина: если для дочерней таблицы существует ref-position Table view, AppSheet может использовать его для рендеринга Related-блока вместо настроенного Deck/Inline view — даже если Referenced table name указывает на полную таблицу, а не на slice.
Решение: удали лишний Table view (ref-position) на дочерней таблице. Если view нужен для другого контекста (например, карточки другой сущности) — замени его на slice + отдельный view, и укажи Referenced table name явно на нужный slice в каждом родительском контексте.
Проверка: после удаления лишнего view перезапусти приложение и убедись что deck отображается корректно.

**Number-тип колонки ключа ref-таблицы отображается с разделителем тысяч**
Симптом: период `202603` отображается как `202 603` в приложении.
Причина: AppSheet автоматически определил тип колонки как Number, который форматируется с разделителями тысяч.
Решение: Data → ref_periods → period → Type → поменять с Number на **Text**. После сохранения значения отображаются без форматирования.

**Valid_If на Ref-колонке не фильтрует dropdown, показывает все значения**
Симптом: формула Valid_If сохранена, но дропдаун всё равно показывает весь справочник.
Причина: Valid_If возвращает Yes/No вместо списка ключей.
Решение: переписать Valid_If так, чтобы он возвращал `SELECT(ref_table[key_column], condition)` — список ключей. Только тогда AppSheet фильтрует dropdown.

Пример неправильной формулы (Yes/No — не фильтрует):
```
AND([_THIS] >= X, [_THIS] <= Y)
```
Пример правильной формулы (List — фильтрует dropdown):
```
SELECT(ref_periods[period], AND(NUMBER([period]) >= X, NUMBER([period]) <= Y))
```

**Branch on a condition — правая панель не открывается**
Симптом: клик на блок Branch в Bot не открывает настройки в правой панели — панель показывает «Automation settings».
Решение: условие вводится через серое поле прямо внутри блока Branch. Клик на это поле открывает Expression Assistant.

**Confirmation Message — синтаксис `<<column>>` не работает**
Симптом: при вводе `<<ANY(settings[col])>>` в поле Confirmation Message — ошибка `Unexpected content: "<"`.
Причина: поле Confirmation Message принимает AppSheet-выражения (Expression Assistant), но не шаблонный синтаксис `<<...>>`.
Решение: использовать `CONCATENATE()`:
```
CONCATENATE("Текст ", ANY(settings[current_period]), " продолжение")
```

**Grouped Action не видит actions других таблиц**
Симптом: в дропдауне Actions у `Grouped: execute a sequence of actions` отображаются только actions той же таблицы.
Причина: Grouped Action ограничен actions своей таблицы.
Решение: использовать `Data: execute an action on a set of rows` — этот тип может вызывать actions любой таблицы через Referenced Table + Referenced Rows + Referenced Action.

**KEY-колонка не скрывается в detail view на writable-таблице**
Симптом: `id` (или другая KEY-колонка) продолжает отображаться в detail view несмотря на `Show? = false`, Column order Manual без `id` в списке, или любые другие попытки скрыть.
Причина: AppSheet не позволяет скрыть KEY-колонку на writable-таблице стандартными методами.
Решение: принять как ограничение AppSheet. Единственный полный обходной путь — перевести таблицу в Read-Only (Update mode), но это отключает редактирование через все формы. Если таблица должна оставаться редактируемой — KEY в view убрать нельзя.

**Branch condition вызывает каскадирование — каждый клик создаёт новый месяц charges**
Симптом: защита от повторного закрытия периода не срабатывает. Каждый клик кнопки создаёт charges ещё на один месяц вперёд.
Причина: Branch проверяет «следующий период после `current_period`». Но `current_period` — VC, пересчитывается синхронно после ForEach. На второй клик «следующий период» всегда пуст, Branch уходит в NO-ветку.
Решение: использовать календарно-привязанное условие (см. Q.10):
```
AND(
  NUMBER(ANY(settings[current_period])) > (YEAR(TODAY()) * 100) + MONTH(TODAY()),
  ISNOTBLANK(SELECT(charges[charge_id], AND(
    [period] = ANY(settings[current_period]),
    [charge_type] = "Авто"
  )))
)
```

**Новая физическая колонка в Sheets не появляется в AppSheet**
После добавления заголовка в Google Sheets колонка не появляется в Data редактора автоматически.
Решение: Data → таблица → ⚙️ → **Regenerate Schema**. После Regenerate проверить тип колонки — AppSheet часто определяет его неверно (например, `Name` вместо `DateTime`).

---

## Section Q — Automation: Bots and Action Sequences

### Q.1 Bot Event Types

AppSheet Bots support three event source types:
- **App (Data change):** triggers on Adds, Deletes, or Updates to a specific table
- **Scheduled:** triggers on a time schedule (hourly, daily, weekly, etc.)
- **Manual trigger:** does NOT exist as a separate event type in AppSheet

To trigger a Bot from a button click, use the **service column pattern** (see Q.2).

### Q.2 Manual Bot Trigger Pattern (Button → Bot)

AppSheet has no "run Bot on button press" event. The idiomatic workaround:

1. Add a **physical DateTime column** (e.g., `last_trigger`) to a single-row service table (e.g., `settings`). Set Show? = false.
2. Create a **technical Action** on `settings` (type: `Data: set the values of some columns in this row`): sets `last_trigger = NOW()`. Set Show? condition = `FALSE`.
3. Create the **user-facing Action** on the working table (type: `Data: execute an action on a set of rows`): Referenced Table = `settings`, Referenced Rows = `SELECT(settings[id], TRUE)`, Referenced Action = the technical action above. Configure Display name, Show_If, Needs confirmation here.
4. Create a **Bot** with Event: Table = `settings`, Updates only, Condition = `[_THISROW_BEFORE].[last_trigger] <> [_THISROW_AFTER].[last_trigger]`.

This ensures the Bot fires exactly once per button press, regardless of other updates to `settings`.

**⚠️ The trigger column must be physical (in Google Sheets), not a Virtual Column.** AppSheet cannot write to VCs via Action, so the Bot would never fire.

**⚠️ Adding a new physical column to a connected Sheet:** add the header in Google Sheets first (text only, no formula, no data), then in AppSheet: Data → table → ⚙️ → Regenerate Schema. After Regenerate, set the correct type manually — AppSheet often guesses wrong (e.g., `Name` instead of `DateTime`).

### Q.3 Bot Event Condition: Before/After Column Values

To detect that a specific column changed (and not just any update to the table), use the `_THISROW_BEFORE` / `_THISROW_AFTER` pattern in the Bot Event Condition field:

```
[_THISROW_BEFORE].[column_name] <> [_THISROW_AFTER].[column_name]
```

These are native AppSheet expressions available only in Bot Event Condition context. Source: Google AppSheet documentation — "Access column values before and after an update" (support.google.com/appsheet/answer/11547057).

### Q.4 Bot Process Steps

Steps are added via **+ Add a step** inside the Process section of a Bot. Each step has a type:

- **Run a task** — sends email, notification, SMS, webhook, creates file, calls script
- **Run a data action** — executes an AppSheet Action on a set of rows
- **Branch on a condition** — conditional YES/NO fork
- **Wait** — pauses for a time condition
- **Call a process** — calls a reusable sub-process
- **Return values** — returns values from a sub-process

**Send a notification** is selected within "Run a task" by clicking the bell icon in the right Settings panel. Turn off "Use default content?" to write a custom Title and Body using expressions in the Title and Body fields (Λ icon).

### Q.5 Branch on a Condition — UI Notes

The Branch step has a counterintuitive UI:
- Clicking the Branch block does NOT open settings in the right panel — the panel shows "Automation settings" instead.
- The condition is entered by clicking the **grey field directly inside the Branch block** — this opens Expression Assistant.
- YES branch = condition is TRUE; NO branch = condition is FALSE.
- Sub-steps are added via the **➕ button under each branch arm**.

### Q.6 Confirmation Message in Actions — Syntax

The Confirmation Message field in Action Behavior accepts AppSheet expressions (opened via Expression Assistant, Λ icon). The template syntax `<<column>>` does NOT work here — it throws `Unexpected content: "<"`.

Use `CONCATENATE()` instead:

```
CONCATENATE("Are you sure? Current period: ", ANY(settings[current_period]), ". Next period: ", INDEX(SORT(SELECT(ref_periods[period], NUMBER([period]) > NUMBER(ANY(settings[current_period])))), 1), ".")
```

### Q.7 `Data: execute an action on a set of rows` — Cross-Table Scope

`Grouped: execute a sequence of actions` only shows actions from the **same table** in its action list. It cannot call actions on other tables.

To execute an action on rows of a different table, use `Data: execute an action on a set of rows`:
- **Referenced Table:** the target table
- **Referenced Rows:** a SELECT expression returning keys from that table
- **Referenced Action:** the action to run on each row

This type can be called from any table and references any other table in the app.

### Q.8 Bot Step: Renaming

To rename a Bot step — click **⋮** (three dots) on the step block in the canvas → **Rename** → enter new name → confirm.

The rename option is on the block itself, not in the right Settings panel.

**⚠️** Default step names (`New step`, `New step 1`, `New step 2 2`, etc.) are not sequential — AppSheet assigns them based on creation order. Always rename steps immediately after creation for readable Bot structure.

### Q.9 Bot Step: Changing Step Type

To change a step's type (e.g., from `Run a task` to `Run a data action`):

1. Click **⋮** on the step block → **Change type** (or "Change step type").
2. Select the new step type from the list.

**⚠️** The **`Run a task ▼`** dropdown button inside the step block changes the *task subtype* (email / notification / webhook / etc.) — it does NOT change the step type. Changing step type requires ⋮ → Change type on the block.

**`Run a data action` → `Set row values`** pattern (write to settings after Bot action):
1. Change step type to `Run a data action`.
2. In the right Settings panel, select **Set row values**.
3. In **Set these column(s)**: add columns and set values via Expression Assistant (Λ icon).

This is the correct way to write `last_close_at = NOW()` or `last_close_message = CONCATENATE(...)` to a settings table from a Bot step.

### Q.10 Anti-Cascade Branch Condition Pattern

**Problem:** A Branch condition that checks for charges in "the next period after `current_period`" fails in a live flow because `current_period` is a Virtual Column — it recalculates immediately after the ForEach creates new charges. By the time the Branch is evaluated on a second click, `current_period` has already advanced, so "next period" is always empty, and the YES-branch (already closed) never fires. Result: each button press creates another month of charges (cascade).

**Solution:** Use a calendar-anchored condition instead:

```
AND(
  NUMBER(ANY(settings[current_period])) > (YEAR(TODAY()) * 100) + MONTH(TODAY()),
  ISNOTBLANK(SELECT(charges[charge_id], AND(
    [period] = ANY(settings[current_period]),
    [charge_type] = "Авто"
  )))
)
```

**Logic:**
- Condition 1: `current_period` has already moved past the calendar month → closure already happened.
- Condition 2: charges for that `current_period` actually exist (guard against manual data anomalies).

**Branch = YES** → period already closed, ForEach must not fire.  
**Branch = NO** → calendar month not yet closed, proceed with ForEach.

**Terminology note:** After a successful closure, `current_period` shifts to the newly created period (it is a VC counting max period in charges). When Branch=YES fires on a repeated click, `current_period` already contains the "new current" period — the one whose charges were just created. The YES-branch message should reference `ANY(settings[current_period])` directly (not "next period").

### Q.11 LINKTOFORM vs LINKTOROW — opening forms

**Critical distinction:**

| Function | Opens form for | Behavior with existing KEY |
|---|---|---|
| `LINKTOFORM("view", "col", val, ...)` | **Creating a new row** | Errors with `There is already a row with the key: '<KEY>'`. Pre-filling the KEY does NOT switch to edit mode. |
| `LINKTOROW([_THISROW], "view")` | **Editing an existing row** | Opens that row in the form. |

**Use cases:**
- `LINKTOFORM` — when you need to create a child record from a parent context (e.g., "add a child to this family" → opens new student form with `family_id` pre-filled).
- `LINKTOROW` — when you need a user to fill ad-hoc fields on the existing row before applying a transaction (e.g., "withdraw this student" → opens form on the existing student row to capture `withdrawal_date`).

**Required for `LINKTOROW` to work cleanly:**
- The target Form view must be on a Slice with `Update mode: Updates only` (not Adds). Otherwise the Slice exposes a `+` button on its views.
- Target row must be a member of the Slice (no Row filter excluding it).

**Source:** discovered in pre-#08.1 §3 when implementing the withdrawal flow. First attempt with `LINKTOFORM("students_withdrawal_form", "student_id", [student_id])` failed with key-conflict error. Switched to `LINKTOROW([_THISROW], "students_withdrawal_form")` — works.

### Q.12 Grouped action with cross-table operations

**Problem:** A single UX action sometimes needs to (a) update columns on the parent row and (b) modify rows in a child table. AppSheet's Grouped action cannot directly call actions defined on other tables.

**Pattern:**

1. Action **A** — `Data: set the values of some columns in this row` on parent table. Position: Hide.
2. Action **B** — `Data: execute an action on a set of rows` on parent table. Referenced table: child table. Referenced rows: `SELECT(child[child_key], [parent_fk] = [_THISROW].[parent_pk])`. Referenced action: a helper action D. Position: Hide.
3. Action **C** — `Grouped` containing [A, B] in order. Position: Hide.
4. Action **D** — `Data: delete this row` (or `set values`) on the **child** table. Position: Hide. Helper called by B.
5. Action **E** — user-facing entry point. Triggers C either directly (if no user input needed) or via a Form Saved event on a Form view (if user input is needed first).

**Why this works:** B is defined on the parent table, so Grouped C can include it; B's `execute on a set of rows` reaches into the child table internally, where D operates on the child row.

**Source:** pre-#08.1 §3, withdrawal action chain (see `ARCHITECTURE.md` §3.12 for the concrete `students__withdrawal_apply` implementation).

### Q.13 Form Saved event as transaction trigger + Confirmation timing

**Pattern for "user confirms intent → fills data → transaction applies":**

1. User-facing action **E** (`App: go to another view within this app`):
   - Target: `LINKTOROW([_THISROW], "form_view")`.
   - `Needs confirmation? = TRUE`.
   - Confirmation Message: dynamic via CONCATENATE.
2. Form view **F** on a Slice (Updates only).
3. **Form Saved** event on F → Action **C** (the Grouped action from Q.12).

**UX flow:**
1. User clicks E → Confirmation dialog (e.g., "Are you sure?").
2. User confirms → form F opens (LINKTOROW reaches the existing row in edit mode).
3. User fills required fields, clicks Save.
4. Form Saved event triggers C → cascade of data operations completes.

**Why Confirmation on E (not on C):**
- Confirmation fires **before** the form opens — user states intent first, then fills details.
- If Confirmation were on C, the user would fill the form, hit Save, and only then see "are you sure?" — wasted effort if they cancel.
- Form-saved actions don't natively support Confirmation in the same way action chains do.

**Side-effect:** the Slice in Q.12/Q.13 will show the warning `Table <slice_name> does not allow new entries` when the form opens. This is informational and harmless — it's the price of `Update mode: Updates only`. Switching to `Updates+Adds` would silence the warning but expose a `+` button in the form, which is worse UX. Leave the warning.

**Source:** pre-#08.1 §3 (withdrawal flow) + §10 (warning analysis).

### Q.14 LABEL trap when toggling Show on the labeled column

**Problem:** A column marked as LABEL is the source of the row's title in inline lists, deck views, and Detail header (when used as Header column). When you uncheck **Show** on that column intending to hide it from the Detail field list, AppSheet **silently re-assigns LABEL to another visible column** (typically the next physical column like `first_name`). Result: the inline list and Detail header start showing only the partial value (e.g., "Артём" instead of "Артём Петров").

**Even worse:** simply re-checking Show on the original column does NOT restore LABEL. AppSheet keeps LABEL on the new captor column, and the editor will not let you turn LABEL back on the original until you first un-check LABEL on the captor.

**Correct sequence to hide a LABEL-bearing column from the field list while keeping the title intact:**

1. Verify which column currently holds LABEL (Data → table → look for the 🏷️ icon).
2. If you want to keep that column as the title source — **leave LABEL on it** and proceed.
3. **Uncheck Show** on that same column. The Header column setting in Detail view continues to render the value (Header column reads the value independently of Show).
4. Verify: open the Detail view of any row — header shows the full computed name, the field list no longer shows the duplicate row.

**If the title breaks (only partial value shown after step 3):**

1. Find which column LABEL jumped to (Data → table → 🏷️).
2. Uncheck LABEL on that captor column.
3. Re-check LABEL on the original column.
4. Verify Show on the original is `false` (your intent), title is correct.

**Source:** discovered 09.05.2026 while cleaning up `_ComputedName` duplication in `students_visible_Detail`. The first attempt (just unchecking Show) silently moved LABEL to `first_name`; the inline list lost surnames; re-checking Show did not restore the state. Resolution required manually moving LABEL back.

### Q.15 Header column renders independently of Show

**Pattern:** A column referenced in **Header column** of a Detail view (UX → Views → `<view>_Detail` → View Options → Header column) is rendered as the large title at the top of each Detail card. **This rendering ignores the Show flag of the column.**

This means: to display a computed name (e.g., `_ComputedName = CONCATENATE([first_name], " ", [last_name])`) only as the header, without duplicating it as a row in the field list:

1. On the column: keep LABEL = ✅ (so deck/inline views also use it as the row title).
2. On the column: set Show = false (suppresses the duplicate row in Detail field list).
3. In Detail view: keep the column listed under Header column.

The header continues to render correctly because Header column has its own pipeline.

**Source:** confirmed 09.05.2026 on `students_visible_Detail` with `_ComputedName`.

---

### Q.16 Form header via Section_Header virtual column

**Problem:** AppSheet form views have no dedicated "form title" or "header text" field. A form's Display name controls the navigation entry/breadcrumb, not a visible header inside the form.

**Pattern:** Use a virtual column of type **Show** with category **Section_Header**. Place it as the **first** column in the form's Column order. The VC's Display name renders as a section header at the top of the form.

| Parameter | Value |
|---|---|
| Column name | `_form_header_<table>` (convention) |
| Type | `Show` |
| Category | `Section_Header` |
| App formula | `""` (empty string) |
| Display name | The header text user should see, e.g. `"Создать/редактировать тариф"` |
| Show? | ✅ true |

**Where to add:** Data → `<table>` → `+` (add virtual column) → set parameters above. Then UX → Views → `<table>_Form` → Column order Manual → put `_form_header_<table>` first.

**Why this works:** `Show` columns of category `Section_Header` are rendered specifically as visual section dividers/headers in form layout. They have no input control and no data binding — purely presentational.

**Use case in this project:** AppSheet does not let two form views (one for Add, one for Edit) coexist on the same table without the system Add action perversely binding to the wrong one (see Q.17). The pragmatic compromise is a single form view with a unified header text such as `"Создать/редактировать <X>"` — same form serves both Add and Edit, header always reads correctly regardless of mode.

**Source:** pre-#08.3, 10.05.2026. Replaces the broken reference in Section F line 122 to a never-written "Section L."

---

### Q.17 Add/Edit form routing — single form view limitation (May 2026)

**Background:** Trying to give Add and Edit different headers by creating two form views (`*_Form` for Add, `*_Form_Edit` for Edit) and routing Edit through a custom action with `LINKTOROW`. The Edit half worked. The Add half failed in every attempted variant.

**What does NOT work** (verified May 2026):

1. **Position = `ref` on `*_Form_Edit`.** Does not stop the system Add from binding to it. AppSheet's `+` button can still select the Edit form when invoked from the deck/table view.
2. **Action type `App: open a form to add a new row` with explicit Target.** In the current editor (May 2026), the **Target form view selector is missing** for this action type. Cannot direct Add to a specific form view this way.
3. **Renaming the Edit form with a `z_` prefix** to push it later alphabetically. Does not help — AppSheet's system Add caches the bound form by internal ID, not by name. After rename, `+` still opens the (now-renamed) Edit form, breadcrumb shows the new name (`z tariffs Form Edit`).
4. **Editing the system Add action's Target.** It is **read-only** in the current editor — cannot be redirected.
5. **Deck view → Behavior → Event Actions.** No "Add" event in the current editor — only `Row Selected` and `Row Swiped`. There is no UI affordance to override what `+` does at the view level.

**Conclusion:** In the current AppSheet editor, **two form views per table cannot be reliably split into Add-only and Edit-only roles** without external Add wiring (and that wiring is itself unavailable, per points 2 and 4).

**Pragmatic resolution:** keep one form view per table. Use a single Section_Header VC (Q.16) with a header text that reads correctly for both Add and Edit — e.g., `"Создать/редактировать тариф"`. Semantics merge, but the UX problem (operator opens form, gets distracted, returns 15 seconds later, cannot tell what they were doing) is resolved.

**Source:** pre-#08.3, 10.05.2026. Three failed paths (LINKTOROW + system Edit override; rename with `z_` prefix; delete-and-recreate) confirmed by strategist after live testing. Recorded so successors don't waste hours rediscovering the limit.

---

### Q.18 ⚠️ CRITICAL — Virtual Columns inside Bot transactions read a frozen snapshot

**The rule:** if a value must be **fresh** at the moment a Bot runs, it MUST be a **physical column**, not a Virtual Column. Inside a bot transaction AppSheet reads VCs (and `SELECT()`/`ANY()` across tables) from a **frozen snapshot** of all tables — that snapshot does NOT recompute during bot execution and can lag behind reality by minutes or hours.

**Symptom that revealed this (project incident, 12-13.05.2026):**

- VC `settings[current_period]` = `INDEX(SORT(SELECT(charges[period], TRUE), TRUE), 1)`.
- Bot `bot_close_period` reads `ANY(settings[current_period])` to decide which period to close.
- Frontend dashboard, Data Explorer, app sync — all showed `current_period = 202605`.
- Bot consistently saw `current_period = 202603` inside its transaction. Created charges for the wrong period (202604 instead of 202606).
- Same bot snapshot Output showed `current_period: 202605` AND `dash_пред_период: 202602` (= current - 1, computed when current was 202603). **Two VCs frozen at different moments inside the same transaction.**
- Trigger: heavy modification of another table (`student_tariffs` wiped+recreated by another bot) the day before. Snapshot got "stuck" on the pre-modification state.

**Confirmed by Gemini:** classical AppSheet bot snapshot caching. VC, SELECT across tables, ANY of another table — all hit the cached snapshot. Even directly inlining the formula (replacing `ANY(settings[current_period])` with the underlying `INDEX(SORT(SELECT(charges[period], TRUE), TRUE), 1)`) **does not help** — `SELECT(charges[...])` reads the same frozen snapshot.

**The canonical fix:**

1. Convert the VC to a **physical column** (add a header to the Google Sheet, Regenerate Schema in AppSheet, drop the old VC). Set correct Type and Show? — `Regenerate Schema` will guess wrong (often `Duration`).
2. Find the action that fires the bot (e.g., the trigger action that writes `NOW()` to the trigger column).
3. Extend that action's **Set columns** to also write the formula into the physical column.
4. Result: the row that triggers the bot's Data Change Event already contains the fresh value when the bot starts — the bot reads it from the event row, no snapshot involved.

**Project example:**

- Trigger action `settings__update_trigger` (writes `last_close_trigger = NOW()`) extended:
  - `last_close_trigger` = `NOW()`
  - `current_period` = `INDEX(SORT(SELECT(charges[period], TRUE), TRUE), 1)`
- VC `current_period` deleted. Physical column with same name added in Sheets.
- All 18+ formulas referencing `settings[current_period]` (`ANY(...)`, dashboard tiles, Valid_if on `charges[period]`, action formulas) automatically rebound to the physical column on Save. No formula edits needed.

**When this matters:**

- ANY bot that needs to read a derived value from another table at runtime.
- Any `SELECT(other_table[col], ...)` inside a bot step formula — same caching issue.
- Especially after another bot has just modified large numbers of rows (the snapshot is more likely to be stale).

**When this does NOT matter:**

- Frontend rendering (VCs recompute on sync).
- Bots that only read columns from the triggering row itself (no `SELECT` across tables).
- Bots triggered by user action where you control the moment of activation.

**Cosmetic side-effect to watch for:** if the physical column is updated **before** the bot's main work (so the work uses fresh value) but the bot also writes a status message **after** doing work that changes the underlying data, the message formula may read the post-work value of the physical column (because the snapshot now contains the new rows the bot just created). In the project, `bot_close_period` writes "Period X closed" using `ANY(settings[current_period])` after the ForEach — X comes out as the **new** max (= closed + 1), not the closed period. Workaround options: compute "closed period" as `current - 1` in the message formula via the YYYYMM-arithmetic IFS pattern; or store the value to a separate physical column before the ForEach.

**Source:** SubGeneral #15, 13.05.2026. Full diagnostic chain in `reports/2026-05-13_soklever-appsheet-report-15.md`. Gemini consultation confirmed the snapshot model.

---

## Section R — Slices and System-Generated Views

### R.1 Each Slice has its own auto-generated Detail/Form/Inline views

When a Slice is created, AppSheet automatically generates a separate set of system views for it: `<slice_name>_Detail`, `<slice_name>_Form`, `<slice_name>_Inline`. These exist **alongside** the system-generated views for the underlying table (`<table>_Detail`, etc.).

**Implication:** if a primary view (e.g., a deck named "ученики") is built on a Slice, then clicking a row opens **`<slice>_Detail`** — not `<table>_Detail`. Editing `<table>_Detail` (Column order, Header column, etc.) has **no effect** on what users see.

**How to identify which Detail view is actually live:**
1. UX → Views → primary navigation → click the entry-point view → **For this data** field shows either a table name or a slice name.
2. If it's a slice, the live Detail view is `<slice>_Detail` under SYSTEM GENERATED.
3. Configure that one (Column order Manual, Header column, etc.).

**Implication for cleanup:** if `<table>_Detail` is no longer reachable from any navigation path (because all entry-points use slices), it becomes a dead view. Candidate for deletion in a cleanup pass.

**Source:** discovered 09.05.2026. Primary view "ученики" was built on slice `students_visible` (introduced in pre-#08.1). All Column order Manual edits to `students_Detail` had been ineffective because `students_visible_Detail` was the live view.

---

### R.2 Slice Update mode = Read-Only blocks system actions in slice-Detail

When a Slice has `Update mode = Read-Only`, AppSheet **hides** the system Edit/Delete actions in the slice's auto-generated Detail view — even when:
- The system action on the underlying table has a correctly-configured `Only if this condition is true` (Show_If) expression.
- The user role would otherwise have edit/delete permission.

The Show_If on the action is **not** the reason the button is missing. The button is suppressed at the slice layer, before Show_If is evaluated.

**Symptom:** "Edit/Delete button is visible when opening a row through the table's system Detail view, but missing when opening the same row through `<slice>_Detail` (e.g., a filter view built on a slice)."

**Fix:** open the slice (Data → Slices → `<slice_name>`) and check **Update mode**. To allow Edit/Delete in the slice-Detail, ensure **Updates** and/or **Deletes** are toggled on (not Read-Only). The Show_If on the system action continues to enforce row-level visibility — security is preserved.

**Note:** In May 2026 (AppSheet 1.000535) the Update mode selector has four mutually-related toggles: **Updates / Adds / Deletes / Read-Only**. Read-Only is the override that disables the other three at the slice level.

**Source:** pre-#08.6, 11.05.2026. Three slices (`charges_filtered`, `payments_filtered`, `expenses_filtered`) had Read-Only set. Edit/Delete were invisible in `*_filtered_Detail` despite proper Show_If on the system actions on `charges`/`payments`/`expenses`. Switching to Updates+Deletes restored visibility. Show_If conditions kept blocking edit/delete on closed-period rows as intended.

---

### R.3 Mobile slice-Detail may suppress Prominent actions even when desktop shows them

**Symptom (May 2026):** in the same Detail view (slice-Detail), on **desktop** the Delete and Edit buttons both render in the top action bar; on **mobile**, only the Edit FAB (bottom-right) is visible — Delete (Position = Prominent) is missing.

**What does not explain it:**
- Slice Update mode: confirmed `Updates + Deletes` on all three slices.
- System action Position: identical (`Edit = Primary`, `Delete = Prominent`) across `charges`, `payments`, `expenses`.
- Slice Actions: identical (`Auto assign`) across all three.

**What partially explains it:** at least one comparable slice in the same app (`payments_filtered`) **does** show Delete on mobile under the same nominal settings. So a configuration difference exists somewhere — but it is not in the Position/Update-mode/Slice-Actions fields visible in the editor.

**Status (11.05.2026): open floating defect.** Root cause not identified. Tested resolution paths and what was learned:

1. **App foregrounding** (screen off / on) — owner reported Delete appearing after the device wakes. Initially suggested a render-on-foreground refresh. Not a reliable resolution: same view reopened later showed Delete missing again.
2. **Full logout/login** on the AppSheet mobile app — **did not help**. The configuration is fully re-fetched after login, so the issue is not a stale client cache.

**Refined pattern (after logout test):** the visibility of Delete on mobile correlates with the **entry route** to the Detail view, not with the slice itself:
- Entering a row through a **filter-view** built on a slice (e.g., `фильтр начислений`, `фильтр расходов`) → Delete missing on mobile (visible on desktop).
- Entering the same row through a **non-filter route** (e.g., payment from the student's card; expense from the expenses menu list) → Delete visible on mobile.

This means the mobile renderer treats Detail-via-filter-slice differently from Detail-via-deck/table or Detail-via-related-inline, even when Position = Prominent. Desktop does not show the asymmetry.

**Untested hypotheses, recorded for future work:**
1. Try `Position = Primary` instead of `Prominent` on the system Delete — see if Primary survives the filter-view route on mobile (puts the icon in the top-right overflow instead of the bottom FAB).
2. Inspect whether the filter-view (`фильтр начислений` etc.) has any view-level Action filtering setting that hides Prominent buttons on small viewports.

**Workaround / accepted state:** desktop is the primary admin workflow. Show_If still enforces the closed-period guard regardless of which actions render. The mobile gap was accepted as a floating defect rather than chased further.

**Source:** pre-#08.6, 11.05.2026. Observed on `charges` and `expenses` filter-view route on mobile; `payments` route from the student card on the same device rendered Delete normally. Full mobile logout/login did not resolve.

---

## AppSheet Expression Reference

| Need | Expression |
|------|-----------|
| Current user's email | `USEREMAIL()` |
| Current user's role | `LOOKUP(USEREMAIL(), "users", "email", "role")` |
| Sum related records | `SUM(SELECT(table[column], [fk] = [_THISROW].[pk]))` |
| Auto-generate ID | `UNIQUEID()` |
| Today's date | `TODAY()` |
| Current datetime | `NOW()` |
| Max of text/enum list | `INDEX(SORT(SELECT(table[col], condition), TRUE), 1)` |
| Current YYYYMM string | `CONCATENATE(YEAR(TODAY()), IF(MONTH(TODAY()) < 10, "0", ""), MONTH(TODAY()))` |
| YYYYMM N days ago | `CONCATENATE(YEAR(TODAY()-N), IF(MONTH(TODAY()-N) < 10, "0", ""), MONTH(TODAY()-N))` |
| YYYYMM as number | `NUMBER([period])` |
| Period >= N days ago | `NUMBER([period]) >= (YEAR(TODAY()-N)*100 + MONTH(TODAY()-N))` |
| Academic year start | Store as fixed Date value in settings sheet — do not compute dynamically |
| Read single-row table | `ANY(settings[column])` or `INDEX(settings[column], 1)` |
| Check role | `LOOKUP(USEREMAIL(), "users", "email", "role") = "ADMIN"` |
| Filter: blank = all | `OR(ISBLANK(ANY(settings[filter])), [column] = ANY(settings[filter]))` |
| Clip value to zero | `IFS(value > 0, value, TRUE, 0)` — MAX(0, value) not supported |
| Compare Ref to Ref | `TEXT([ref_column]) = TEXT(ANY(settings[ref_filter]))` |
| Pre-fill form (create new row) | `LINKTOFORM("FormName", "col1", [val1], "col2", [val2])` |
| Open existing row in form (edit) | `LINKTOROW([_THISROW], "FormName")` |
| Label for deck (VC) | `CONCATENATE([fk], ": ", [display_field])` — type Text, Label = ✅ |
| Previous YYYYMM period | `IFS(NUMBER(RIGHT(ANY(settings[current_period]),2))>1, NUMBER(LEFT(ANY(settings[current_period]),4))*100+NUMBER(RIGHT(ANY(settings[current_period]),2))-1, TRUE, (NUMBER(LEFT(ANY(settings[current_period]),4))-1)*100+12)` |
| Dynamic view block header | Set Display name in view's Display section to formula using `ANY(settings[col])` |
| Display name of Related block | Data → parent table → Related X → pencil → Display → Display name |
| Show ref field value in form | Use VC type Text (not Show) with App formula `[ref_id].[field]` — Show type does not compute for new rows |
| Filter Ref dropdown by condition | `SELECT(ref_table[key_col], condition_on_ref_rows)` — use in Valid_If |
| Convert Text key to number for arithmetic | `NUMBER([text_ref_key])` — required after Enum→Ref conversion |
| Initial value = current period | `ANY(settings[current_period])` |
| Detect column changed in Bot event | `[_THISROW_BEFORE].[col] <> [_THISROW_AFTER].[col]` — Bot Event Condition only |
| Next period from ref_periods | `INDEX(SORT(SELECT(ref_periods[period], NUMBER([period]) > NUMBER(ANY(settings[current_period])))), 1)` |
| Dynamic confirmation message | `CONCATENATE("Text ", ANY(settings[col]), " more text")` — use instead of `<<col>>` syntax |
| Regex pattern match | **NOT SUPPORTED in AppSheet.** No `MATCHES`, `ISMATCH`, `REGEXMATCH` — all are Google Sheets/Apps Script functions, not AppSheet. Verified 12.05.2026 (birth_date text format attempt) + Gemini confirm. |
| Validate date-formatted text (e.g. ДД.ММ.ГГГГ) | Canonical AppSheet way: `OR(ISBLANK([col]), AND(LEN([col])=10, ISNOTBLANK(DATE(SUBSTITUTE([col], ".", "/")))))`. Depends on app **Locale** (Settings → Information → Regional must be ru/ua for DMY parsing). |
| Date-Text round-trip | `DATE(SUBSTITUTE([text_col], ".", "/"))` — converts validated ДД.ММ.ГГГГ text back to Date type (VC or initial value). |
| Mobile-friendly date input (no calendar picker) | Native Date type forces calendar picker (terrible for birthdates 10+ years back). Three workarounds: (1) keep Type=Text + validation above + Description hint; (2) three Enum columns Day/Month/Year, assemble via `DATE(CONCATENATE([Month],"/",[Day],"/",[Year]))`; (3) Display→Input mode — `Buttons`/`Stack` rarely affects Date picker, AppSheet uses native OS components. |
| Bot-written timestamp column — no Initial value | Columns written exclusively by Bot steps (Set row values with NOW()) should have **empty Initial value**. AppSheet auto-fills Initial value = NOW() on DateTime columns after Regenerate Schema — manually clear it. Otherwise: if a new row is ever inserted via form/manual, it logs row-creation time, not last-event time → semantic collision with bot writes. Verified 12.05.2026 on `settings[last_event_at]` (#08). |
| Auto-assigned actions on dashboard slices | When adding a new Prominent action on a base table (e.g. `settings`), AppSheet **auto-assigns** it to ALL detail views built on slices of that table. On a dashboard composed of slice-based detail tiles (e.g. `settings_dash_открытый`, `settings_dash_долги`, …) the button appears on every tile. Fix: in each slice's **Slice Actions**, remove the action from Auto assign (set to explicit list or empty). Discovered 12.05.2026 (#08 §4.4 / §4.5). |

---

## Architecture Notes

- Sheets that are purely computed (dashboard, debts) should NOT be connected to AppSheet
- `settings` table: single-row service table for app-wide parameters (current period, filter values, academic year start)
- Period format: YYYYMM as Text (e.g., "202604" = April 2026) — **Text, not Number**, to avoid thousands-separator display issues
- Reference tables (e.g., `expense_categories`, `ref_periods`, `ref_classes`) should be Ref columns, not Enum — enables future extensibility and single-source-of-truth maintenance
- AppSheet auto-creates `Related X` virtual columns (List type) when Ref + "Is a part of?" is configured
- Filter by period (YYYYMM Text) is simpler and more reliable than filter by date range for school billing use case
- Two-entry-point navigation: PRIMARY for quick add, MENU for filtered analytics
- Inline related records: use Slices + ref-position Deck views; point `Related X` Referenced table name to the slice
- Deck requires: (1) Label column on source table, (2) Primary/Secondary/Summary filled, (3) Referenced table name pointing to slice
- Dashboard multi-block: one slice + one Detail view per block; Display name of view = dynamic formula via `ANY()`
- Role-based filtering: add `OR(LOOKUP(...) = "ADMIN", [created_by] = USEREMAIL())` to slice Row filter — not to Security filter, to preserve ADMIN access
- Referenced table name accepts only tables and slices — not views. Always use a slice as intermediary for custom Related block rendering.
- A loner ref-position Table view on a child table can override deck rendering in parent cards even when Referenced table name points to the full table. Keep ref-position views minimal and intentional.
- After Enum→Ref refactor: delete auto-generated Detail/Form views for reference tables; hide/delete auto-generated `View Ref (col)` actions — they are not useful for end users.
- Manual Bot triggers require a physical service column (DateTime, hidden) in a single-row table. Writing `NOW()` to it via Action fires the Bot. Virtual columns cannot be used as triggers — Actions cannot write to VCs.
- `Grouped: execute a sequence of actions` is limited to actions on the same table. Use `Data: execute an action on a set of rows` to call actions across tables.
- Confirmation Message field uses Expression Assistant (Λ icon) and accepts AppSheet expressions. Template syntax `<<col>>` is not supported — use `CONCATENATE()` instead.
- **KEY column in a detail view on a writable table cannot be hidden** via `Show? = false`, Column order Manual (empty list), or any other standard method. This is an AppSheet limitation. If the KEY column appearing in the view is unacceptable, the only workaround is to make the table Read-Only (but this disables all editing).
- **Bot step renaming:** ⋮ on the step block → Rename. Not in the right panel.
- **Bot step type change:** ⋮ on the step block → Change type. The `Run a task ▼` dropdown inside the block changes task subtype only.
- **Anti-cascade Branch:** never check "next period" in a Bot that shifts `current_period` via ForEach — use calendar-anchored condition instead (see Q.10).
