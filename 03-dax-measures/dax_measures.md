# DAX Measures Documentation

Documentation for the key DAX measures of the **School Performance Analytics** project.

All measures are stored in a dedicated service table `_Measures`
for convenient navigation and separation from data tables.

---

## 1. Basic Metrics

### `Average Grade`

```dax
Average Grade =
DIVIDE(
    SUM(grades[grade_value]),
    DISTINCTCOUNT(grades[grade_id])
)
```

**What it does:** calculates the average grade as the sum of all grades
divided by the count of unique grade IDs.

**Why DISTINCTCOUNT instead of plain AVERAGE:** protects against
double-counting when JOIN operations create duplicate `grade_id` values.
This is a more robust approach for production models.

**Filter context:** responds to all active filters (class, subject, period, date).

---

### `Total Grades`

```dax
Total Grades = COUNTROWS(grades)
```

**What it does:** counts the total number of rows in the `grades` fact table.

**Usage:** base metric for KPI cards and as a denominator in other
calculations (e.g., in `Neg Grades Prop`).

---

## 2. Time Intelligence

### `Average Grade YTD`

```dax
Average Grade YTD = TOTALYTD([Average Grade], 'Calendar'[Date])
```

**What it does:** computes the average grade cumulatively from the start
of the calendar year to the current date in context.

**Technical detail:** the `TOTALYTD` function is shorthand for a
`CALCULATE + DATESYTD` construction. Requires a dedicated `Calendar` table.

**Business value:** allows tracking how the average grade changes as
grades accumulate throughout the year.

---

### `Average Grade MoM%`

```dax
Average Grade MoM% =
VAR __PREV_MONTH =
    CALCULATE(
        [Average Grade],
        DATEADD('Calendar'[Date], -1, MONTH)
    )
RETURN
    DIVIDE([Average Grade] - __PREV_MONTH, __PREV_MONTH)
```

**What it does:** percentage change in average grade compared to the previous month.

**Logic breakdown:**
- `VAR __PREV_MONTH` — variable that stores the previous month's average
- `DATEADD(..., -1, MONTH)` — shifts the date context one month back
- `DIVIDE` — safe division (returns BLANK instead of error on division by zero)

**Business value:** shows the dynamic: is performance growing or declining month over month.

---

### `Previous Grade`

```dax
Previous Grade =
CALCULATE (
    DIVIDE (
        SUM ( grades[grade_value] ),
        DISTINCTCOUNT ( grades[grade_id] )
    ),
    DATEADD ( Calendar[Date], -1, MONTH ),
    REMOVEFILTERS ( Calendar[Year_Month] )
)
```

**What it does:** calculates the average grade for the previous month,
independent of the current filter context.

**Why `REMOVEFILTERS(Calendar[Year_Month])` is needed:** if a `Year_Month`
filter is active in the report, it constrains data to the current month —
and `DATEADD` cannot "step back" to the previous month. Removing this
filter allows correct backward navigation in time.

**Usage:** as an intermediate measure for analysis and comparison.
Allows showing "January average" next to "December average" in the same table.

---

## 3. Filter Context (CALCULATE demonstration)

### `Neg Grades Count`

```dax
Neg Grades Count =
CALCULATE (
    COUNTROWS ( grades ),
    grades[grade_value] < 6
)
```

**What it does:** counts grades below 6 points (negative on a 12-point scale).

**How `CALCULATE` works:** the function **modifies filter context** by
adding the condition `grades[grade_value] < 6` to all other active filters.
This is a classic demonstration of CALCULATE's power — without it, we
would need to create a calculated column.

**Business value:** one of the key KPIs for identifying students who need
additional attention (HW11).

---

### `Neg Grades Prop`

```dax
Neg Grades Prop =
DIVIDE (
    [Neg Grades Count],
    [Total Grades]
)
```

**What it does:** calculates the proportion of negative grades to the total count.

**Why use measures instead of SUM/COUNTROWS:** both arguments are
**references to other measures**, which makes the formula readable and
modular. If `Neg Grades Count` logic changes, `Neg Grades Prop` updates automatically.

**Format:** the value is displayed as a percentage in visuals
(configured in Format → Percentage).

---

## 4. Grade Type Analysis

### `Exam Average Grade`

```dax
Exam Average Grade =
CALCULATE (
    DIVIDE (
        SUM ( grades[grade_value] ),
        DISTINCTCOUNT ( grades[grade_id] )
    ),
    grades[grade_type] = "exam",
    REMOVEFILTERS ( grades[grade_type] )
)
```

**What it does:** calculates the average grade for "exam" type grades only,
regardless of active filters on grade type.

**Logic breakdown:**
- Inner `DIVIDE` — replicates the `Average Grade` formula
- `grades[grade_type] = "exam"` — adds a filter on grade type
- `REMOVEFILTERS ( grades[grade_type] )` — removes external filters on
  grade type so the measure always shows "exam", even if the report has
  another type filter

**Why this construction:** allows showing multiple grade-type averages
side-by-side in one table — exam, test, homework, etc. — and they all work
independently of each other.

> **Note:** the same pattern is used to create analogous measures:
> - `Homework Average Grade`
> - `Project Average Grade`
> - `Quiz Average Grade`
> - `Test Average Grade`
>
> All use the same logic; only the `grades[grade_type]` value changes.

---

## DAX Functions Overview

| Function | Category | Where used |
|---|---|---|
| `SUM` | Aggregation | Average Grade, Exam Average Grade |
| `DISTINCTCOUNT` | Aggregation | Average Grade, Exam Average Grade |
| `COUNTROWS` | Aggregation | Total Grades, Neg Grades Count |
| `DIVIDE` | Safe division | Average Grade, Neg Grades Prop, MoM% |
| `CALCULATE` | Filter context | Neg Grades Count, Previous Grade, Exam Average Grade |
| `TOTALYTD` | Time intelligence | Average Grade YTD |
| `DATEADD` | Time intelligence | Average Grade MoM%, Previous Grade |
| `REMOVEFILTERS` | Filter context | Previous Grade, Exam Average Grade |
| `VAR / RETURN` | Structure | Average Grade MoM% |

---

## Measure Design Philosophy

1. **Modular** — complex measures use simple ones as building blocks
   (e.g., `Neg Grades Prop` references both `Neg Grades Count` and `Total Grades`).
2. **Safe** — `DIVIDE` is used everywhere instead of the `/` operator
   to avoid division-by-zero errors.
3. **Explicit Filter Context** — where competing filters need to be removed,
   `REMOVEFILTERS` is used explicitly.
4. **Variables** — for repeated calculations (like previous month),
   `VAR` is used for readability and performance.
