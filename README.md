# Power BI – Data Modification, Measures & Data Model

> **Author:** Lesana Beňová
> **Context:** CSOB Data Quality Dashboard & PBI Academy

## Introduction

**Power BI** is a business analytics platform by Microsoft that enables users to connect to various data sources, transform and clean data, build data models, and create interactive dashboards and reports. This document covers the key concepts of **Power Query** (data transformation), **DAX measures** (calculations), and **data modeling** (star schema) used in Power BI development.

---

## Table of Contents

- [Power Query](#power-query)
  - [Basic Data Modifications](#basic-data-modifications)
  - [Column Quality & Distribution](#column-quality--distribution)
  - [Merge Queries (+ Columns)](#merge-queries--columns)
  - [Append Queries (+ Rows)](#append-queries--rows)
  - [Transpose](#transpose)
  - [Pivot (Rows to Columns)](#pivot-rows-to-columns)
  - [Unpivot (Columns to Rows)](#unpivot-columns-to-rows)
  - [Split Column](#split-column)
  - [Calendar / Date Table](#calendar--date-table)
  - [Custom Column Examples](#custom-column-examples)
- [Data Model](#data-model)
  - [Star Schema](#star-schema)
  - [Relationships](#relationships)
  - [Reference Tables](#reference-tables)
- [DAX and Measures](#dax-and-measures)
  - [What Is a Measure?](#what-is-a-measure)
  - [COUNTROWS](#1-countrows)
  - [SUMX](#2-sumx)
  - [CALCULATE](#3-calculate)
  - [FILTER ALL (Special Measure)](#4-filter-all-special-measure)

---

## Power Query

Navigate to **Home → Transform Data** (pencil icon) to open the Power Query Editor, where all data corrections and transformations take place.

### Basic Data Modifications

Right-click on a column header to access the following options:

- **Replace Values** – find and replace values within a column
- **Duplicate / Delete Column** – create a copy or remove a column entirely
- **Remove Duplicates** – eliminate duplicate rows based on the selected column
- **Remove Errors** – remove rows that contain errors
- **Change Data Type** – convert the column to a different data type (e.g., text, number, date)
- **Pivot / Unpivot** – reshape data between row and column orientations

Other useful actions:

- **Promote first row as headers** – use the first row of data as column headers (button in the ribbon)
- **Advanced Editor** – view and edit the full **M language** code behind all transformation steps in a query

> **Tip:** ID columns should always be set to **Text** type — you never need to perform mathematical operations on identifiers.

### Column Quality & Distribution

Go to **View** and enable the **Column Quality** and **Column Distribution** checkboxes. This displays statistics for each column, including the percentage of *valid data*, *errors*, and *empty values* — a quick way to assess imported data quality.

---

### Merge Queries (+ Columns)

Use **Merge Queries** to add columns from one table to another (similar to a SQL JOIN).

- Typically used to enrich a **fact table** (e.g., customers, payments) with values from a **dimension table** (e.g., lookup/reference values).
- A **relationship** between the tables is required — both tables must share a common column (e.g., an ID field).
- Choose the appropriate join type: **Left**, **Right**, **Inner**, **Full Outer**, etc.

### Append Queries (+ Rows)

Use **Append Queries** to stack rows from one table below another (similar to SQL `UNION`).

- The tables should have **matching column names** for the append to work correctly.

> 📺 [YouTube – Merge & Append Queries](https://www.youtube.com/watch?v=VaOhNqNtGGE)

---

### Transpose

Transpose swaps **rows and columns** in a table. After transposing, you typically need to promote the first row to headers.

### Pivot (Rows to Columns)

- Converts **rows into columns**.
- Select the column you want to pivot (e.g., *Year*) and summarize based on numeric values.

### Unpivot (Columns to Rows)

- Converts **columns into rows** — resulting in more rows and fewer columns.
- Select the columns you want to *keep* (e.g., *City*), then choose **Unpivot Other Columns**.

> 📺 [YouTube – Pivot & Unpivot](https://www.youtube.com/watch?v=113_LHiVqGA)

### Split Column

Separates data within a cell into multiple columns using a delimiter.

*Example:* `21/10/2020` split by `/` produces three columns: `21`, `10`, `2020`.

---

### Calendar / Date Table

**Every report or analysis should include a date/time dimension.** You can generate a calendar table using a blank query and the following M code:

```m
let fnDateTable = (StartDate as date, EndDate as date, FYStartMonth as number) as table =>
  let
    DayCount = 1 + Duration.Days(Duration.From(EndDate - StartDate)),
    Source = List.Dates(StartDate, DayCount, #duration(1, 0, 0, 0)),
    TableFromList = Table.FromList(Source, Splitter.SplitByNothing()),
    ChangedType = Table.TransformColumnTypes(TableFromList, {{"Column1", type date}}),
    RenamedColumns = Table.RenameColumns(ChangedType, {{"Column1", "Date"}}),
    InsertYear = Table.AddColumn(RenamedColumns, "Year", each Date.Year([Date]), type text),
    InsertYearNumber = Table.AddColumn(RenamedColumns, "YearNumber", each Date.Year([Date])),
    InsertQuarter = Table.AddColumn(InsertYear, "QuarterOfYear", each Date.QuarterOfYear([Date])),
    InsertMonth = Table.AddColumn(InsertQuarter, "MonthOfYear", each Date.Month([Date]), type text),
    InsertDay = Table.AddColumn(InsertMonth, "DayOfMonth", each Date.Day([Date])),
    InsertDayInt = Table.AddColumn(InsertDay, "DateInt", each [Year] * 10000 + [MonthOfYear] * 100 + [DayOfMonth]),
    InsertMonthName = Table.AddColumn(InsertDayInt, "MonthName", each Date.ToText([Date], "MMMM"), type text),
    InsertCalendarMonth = Table.AddColumn(InsertMonthName, "MonthInCalendar", each (try(Text.Range([MonthName], 0, 3)) otherwise [MonthName]) & " " & Number.ToText([Year])),
    InsertCalendarQtr = Table.AddColumn(InsertCalendarMonth, "QuarterInCalendar", each "Q" & Number.ToText([QuarterOfYear]) & " " & Number.ToText([Year])),
    InsertDayWeek = Table.AddColumn(InsertCalendarQtr, "DayInWeek", each Date.DayOfWeek([Date])),
    InsertDayName = Table.AddColumn(InsertDayWeek, "DayOfWeekName", each Date.ToText([Date], "dddd"), type text),
    InsertWeekEnding = Table.AddColumn(InsertDayName, "WeekEnding", each Date.EndOfWeek([Date]), type date),
    InsertWeekNumber = Table.AddColumn(InsertWeekEnding, "Week Number", each Date.WeekOfYear([Date])),
    InsertMonthnYear = Table.AddColumn(InsertWeekNumber, "MonthnYear", each [Year] * 100 + [MonthOfYear] * 1),
    InsertQuarternYear = Table.AddColumn(InsertMonthnYear, "QuarternYear", each [Year] * 100 + [QuarterOfYear] * 1),
    ChangedType1 = Table.TransformColumnTypes(InsertQuarternYear, {
      {"QuarternYear", Int64.Type}, {"Week Number", Int64.Type}, {"Year", Int64.Type},
      {"MonthnYear", Int64.Type}, {"DateInt", Int64.Type}, {"DayOfMonth", Int64.Type},
      {"MonthOfYear", Int64.Type}, {"QuarterOfYear", Int64.Type},
      {"MonthInCalendar", type text}, {"QuarterInCalendar", type text}, {"DayInWeek", Int64.Type}
    }),
    InsertShortYear = Table.AddColumn(ChangedType1, "ShortYear", each Text.End(Text.From([Year]), 2), type text),
    AddFY = Table.AddColumn(InsertShortYear, "FY", each "FY" & (if [MonthOfYear] >= FYStartMonth then Text.From(Number.From([ShortYear]) + 1) else [ShortYear])),
    InsertMonthOfQuarter = Table.AddColumn(AddFY, "MonthOfQuarter", each if List.Contains({1, 4, 7, 10}, [MonthOfYear]) then 1 else (if List.Contains({2, 5, 8, 11}, [MonthOfYear]) then 2 else 3)),
    InsertWorkingDay = Table.AddColumn(InsertMonthOfQuarter, "WorkingDay", each if List.Contains({0, 6}, [DayInWeek]) then "WorkingDay" else "Weekend")
  in
    InsertWorkingDay
in
  fnDateTable
```

**Parameters:**

| Parameter | Description |
|---|---|
| `StartDate` | First date in the calendar |
| `EndDate` | Last date in the calendar |
| `FYStartMonth` | Starting month of the fiscal year (1 = January, 2 = February, etc.) |

> **Important:** When converting date values, follow the order **Number → Text → Date** to maintain the correct format. Avoid blank values in date columns.

---

### Custom Column Examples

Add a custom column via **Add Column → Custom Column**:

**Create a date from Year and Month columns:**

```m
= Date.EndOfMonth(#date([Year], [Attribute], 1))
```

**Convert `month_year` text (format `yyyyMM`) to a date:**

```m
= Date.FromText([month_year], "yyyyMM")
```

**Convert `month_year` to a short text format (e.g., `"Mar 20"`):**

```m
= Date.ToText(Date.FromText([month_year], [Format = "yyyyMM"]), [Format = "MMM yy"])
```

**Create a Year-Quarter key (e.g., `200603` for Q3 2006):**

```m
= [column_year] & "0" & [column_quarter]
```

---

## Data Model

### Star Schema

A **star schema** is the recommended data model structure in Power BI. It consists of:

- **One fact table** — contains all transactional data (e.g., customers, orders, sales amounts).
- **Multiple dimension tables** — contain unique reference/lookup values (e.g., product list, customer list, store list).
- Sometimes the **Calendar table** also acts as a fact table.

**Relationships** between tables should be:

- **1-to-Many (1:N)** — one record in the dimension table relates to many records in the fact table.
- **Single direction** filter flow — avoid bidirectional filtering, as it can cause issues with measures.

```
Dim_Date
    |
    | N : 1
Dim_Product ——— Fact_Sales ——— Dim_Customer
                    |
                    |
                Dim_Store
```

*Examples:*
- 1 manager manages 1 department, but 2 managers should not manage the same department.
- 1 customer can have 5 orders.

### Relationships

- Right-click a table in the model view → **Manage Relationships** to configure.
- **Avoid connecting two fact tables directly** — a Many-to-Many (M:M) relationship is not effective. You cannot place columns from two fact tables together in a visual; use **measures** instead.

### Reference Tables

In Power Query, right-click a table → **Reference** to create an indirect relationship to another query without duplicating the table. This is useful when you need a different version of the same table for a new relationship.

---

## DAX and Measures

### What Is a Measure?

A **DAX measure** is a formula written in the DAX language that dynamically recalculates based on the filters applied in the report (slicers, visuals, date selections).

Key characteristics:

- A measure is **not a column** — it is a mathematical operation that always returns **a single aggregated value** (e.g., SUM, AVG, COUNT).
- Measures interact with all visuals on a page (or across pages), recalculating every time a filter changes.
- Always use **aggregated expressions** — `SUM()`, `AVERAGE()`, `COUNT()`, etc.

**Example — Year-to-Date calculation:**

```dax
Value s DPH meritko 2 = SUM(table1[Value s DPH])

Value YTD =
CALCULATE(
    [Value s DPH meritko 2],
    DATESYTD('Calendar'[Date])
)
```

> `DATESYTD('Calendar'[Date])` returns all dates from the beginning of the year up to the current date in context.

---

### 1. COUNTROWS

Counts the number of rows in a table.

```dax
measure = COUNTROWS(table)
```

---

### 2. SUMX

Iterates over a table row by row, evaluates an expression for each row, and returns the sum.

**Calculate total sales amount:**

```dax
Sales Amount = SUMX('Sales', 'Sales'[Quantity] * 'Sales'[Net Price])
```

**Calculate the difference in days between two dates:**

```dax
date diff = SUMX("table1", DATEDIFF("table1"[start_date], "table1"[end_date], DAY))
```

**With an extra day added (if necessary):**

```dax
date diff = SUMX("table1", DATEDIFF("table1"[start_date], "table1"[end_date], DAY) + 1)
```

---

### 3. CALCULATE

Evaluates an expression in a modified filter context. Supports various logical operations (`IF`, `SUM`, `AVG`, etc.).

**Sum with GROUP BY to handle duplicates:**

```dax
Measure = CALCULATE(
    SUM('SQL_TEST_SALARY_HISTORY (2)'[SALARY]),
    GROUPBY('UNIF employee salary ciselnik', 'UNIF employee salary ciselnik'[ID zamestnanca mzdy])
)
```

**Sum with a filter condition:**

```dax
value = CALCULATE([Sales Amount], table[city] = "IT")
```

---

### 4. FILTER ALL (Special Measure)

A measure using `FILTER(ALL(...))` ignores any filters applied in the visual and always calculates based on the specified condition. Unlike a basic `CALCULATE` filter, this measure is **not adaptive** to slicer or visual context.

```dax
measure 3 = CALCULATE(
    [Sales Amount],
    FILTER(ALL(table1), table1[city] = "IT")
)
```

> **Use case:** When you need a fixed reference value (e.g., total sales for a specific city) regardless of what filters the user applies on the dashboard.

---

*Generated from internal PBI Academy training notes.*
