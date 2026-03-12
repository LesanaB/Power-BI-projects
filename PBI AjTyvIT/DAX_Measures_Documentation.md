# DAX Measures Documentation
## Sales Analytics Dashboard - Complete Measure Reference

---

## Overview
This document provides comprehensive documentation for all DAX measures used in the sales analytics model. These measures calculate key performance indicators including revenue, sales plans, and comparative analysis across time periods and business segments.

---

## Core Measures

### 1. Plan
**Purpose:** Calculates the total sales quota amount for the selected employee(s).

```dax
Plan = CALCULATE(
    SUMX(FactSalesPlan, FactSalesPlan[SalesAmountQuota]), 
    FILTER(DimEmployee, DimEmployee[EmployeeKey])
)
```

**Explanation:** 
- Sums all sales quota amounts from the FactSalesPlan table
- Filters results by the selected employee key from DimEmployee
- Used as a baseline for performance comparisons

**Dependencies:** FactSalesPlan table, DimEmployee table

---

### 2. Plan to Today
**Purpose:** Shows year-to-date (YTD) sales plan amount up to the current date.

```dax
Plan to today = CALCULATE(
    [Plan],
    DATESYTD(Calendar1[Date])
)
```

**Explanation:**
- Uses the existing [Plan] measure as the base calculation
- Applies DATESYTD filter to show only data from the start of the year through today
- Useful for tracking plan progress during the current year

**Dependencies:** [Plan] measure, Calendar1 table with Date column

---

### 3. Year Plan
**Purpose:** Calculates the full-year sales plan total regardless of quarter selection.

```dax
year_plan = CALCULATE(
    [Plan],
    ALL(Calendar1[QuarterOfYear])
)
```

**Explanation:**
- Takes the [Plan] measure and removes any quarterly filters
- Shows the complete annual plan target
- Useful for percentage calculations and performance ratios

**Dependencies:** [Plan] measure, Calendar1 table

---

## Plan Price Measures

### 4. Plan Price
**Purpose:** Calculates the average or total unit price from reseller sales for the selected employee.

```dax
plan_cena = CALCULATE(
    SUMX(FactResellerSales, FactResellerSales[UnitPrice]), 
    FILTER(DimEmployee, DimEmployee[EmployeeKey])
)
```

**Explanation:**
- Sums all unit prices from the FactResellerSales table
- Filters by selected employee
- Represents the pricing component of the sales plan

**Dependencies:** FactResellerSales table, DimEmployee table

---

### 5. Plan Amount × Price
**Purpose:** Multiplies the plan quantity by the plan price to show revenue impact.

```dax
plan_a_cena = [Plan] * [plan_cena]
```

**Explanation:**
- Combines plan quantity with pricing information
- Shows the planned revenue value
- Used for financial forecasting and planning

**Dependencies:** [Plan] measure, [plan_cena] measure

---

## Revenue Measures

### 6. Revenue
**Purpose:** Calculates total revenue from all reseller sales transactions.

```dax
Revenue = SUMX(
    FactResellerSales, 
    FactResellerSales[UnitPrice] * FactResellerSales[OrderQuantity]
)
```

**Explanation:**
- Multiplies unit price by order quantity for each transaction
- Sums all transactions in the selected period
- Base measure for all revenue comparisons

**Dependencies:** FactResellerSales table

---

### 7. Revenue LY (Last Year)
**Purpose:** Shows revenue from the same period in the previous year for year-over-year comparison.

```dax
Revenue LY = CALCULATE(
    [Revenue],
    SAMEPERIODLASTYEAR(Calendar1[Date])
)
```

**Explanation:**
- Takes the [Revenue] measure and shifts it to the same period last year
- Enables year-over-year growth analysis
- Automatically adjusts for whatever time period is selected

**Dependencies:** [Revenue] measure, Calendar1 table with Date column

---

### 8. Revenue Reseller
**Purpose:** Calculates revenue attributed specifically to reseller transactions.

```dax
Revenue reseller = CALCULATE(
    SUMX(
        FactResellerSales, 
        FactResellerSales[UnitPrice] * FactResellerSales[OrderQuantity]
    ),
    FILTER(FactResellerSales, FactResellerSales[ResellerKey])
)
```

**Explanation:**
- Filters revenue to include only reseller sales
- Excludes direct sales and other channel types
- Useful for channel-specific reporting

**Dependencies:** FactResellerSales table

---

## Comparative and Analytical Measures

### 9. Revenue Share of Reseller
**Purpose:** Calculates the percentage of total revenue contributed by a specific reseller.

```dax
Revenue share of reseller = [Revenue reseller] / 
CALCULATE([Revenue reseller], ALL(DimPartner[ResellerName]))
```

**Explanation:**
- Divides reseller-specific revenue by total reseller revenue (all partners)
- Shows market share or contribution percentage per reseller
- Removes partner filter in denominator to get company-wide total

**Dependencies:** [Revenue reseller] measure, DimPartner table

---

### 10. Revenue Share Top 20
**Purpose:** Calculates the revenue share within the current selection context.

```dax
revenue share top 20 = [Revenue reseller] / 
CALCULATE([Revenue reseller], ALLSELECTED())
```

**Explanation:**
- Divides reseller revenue by total revenue in the current selection
- ALLSELECTED maintains external filters but removes internal row context
- Shows what percentage this reseller represents in the filtered dataset

**Dependencies:** [Revenue reseller] measure

---

### 11. Revenue vs Plan
**Purpose:** Compares actual revenue against the planned sales target.

```dax
Revenue vs Plan = [Revenue] / [Plan]
```

**Explanation:**
- Ratio of actual revenue to planned sales quota
- Values >1.0 indicate exceeding the plan
- Values <1.0 indicate underperformance against plan
- Can be formatted as percentage (multiply by 100)

**Dependencies:** [Revenue] measure, [Plan] measure

---

### 12. Revenue vs Revenue LY (Last Year)
**Purpose:** Shows year-over-year growth rate.

```dax
Revenue vs Revenue LY = IF([Revenue LY] = 0, BLANK(), 
[Revenue] / [Revenue LY])
```

**Explanation:**
- Divides current revenue by last year's revenue
- IF statement handles division by zero when there was no prior-year sales
- Results >1.0 indicate growth; <1.0 indicate decline
- BLANK() prevents error display when comparing against zero

**Dependencies:** [Revenue] measure, [Revenue LY] measure

---

## Summary Table

| Measure | Type | Purpose |
|---------|------|---------|
| Plan | Quota | Sales target for employee |
| Plan to today | Quota | Year-to-date plan progress |
| Year Plan | Quota | Full annual plan target |
| Plan Price | Pricing | Unit price component |
| Plan Amount × Price | Planning | Revenue impact of plan |
| Revenue | Actual | Total realized revenue |
| Revenue LY | Comparative | Prior-year revenue |
| Revenue Reseller | Segment | Revenue by channel |
| Revenue Share of Reseller | Analytical | Market share percentage |
| Revenue Share Top 20 | Analytical | Share within selection |
| Revenue vs Plan | KPI | Plan achievement ratio |
| Revenue vs Revenue LY | KPI | Year-over-year growth |

---

## Implementation Notes

### Filter Propagation
- Measures respect calendar, employee, and partner filters applied in visuals
- Use ALL() to ignore specific columns when calculating totals
- Use ALLSELECTED() to preserve external context

### Performance Optimization
- Consider materialized calculations for frequently used ratios
- SUMX operations can be expensive with large tables—monitor query performance
- Use column relationships instead of FILTER when possible

### Best Practices
1. All percentage measures should be formatted as percentage (0.00%)
2. Revenue measures should be formatted as currency with 2 decimal places
3. Plan comparison ratios can be shown as indices (×100) for easier interpretation
4. Consider adding variance measures [Revenue - Plan] for absolute difference

---

## Related Tables

**FactSalesPlan:** Contains quota and planning data
- SalesAmountQuota (Primary column used)
- EmployeeKey (For filtering)

**FactResellerSales:** Contains actual sales transactions
- UnitPrice (Pricing)
- OrderQuantity (Quantity)
- ResellerKey (Channel filtering)

**Calendar1:** Contains date and period information
- Date (For time-based calculations)
- QuarterOfYear (For quarterly filtering)

**DimEmployee:** Contains employee dimension
- EmployeeKey (Primary key)

**DimPartner:** Contains reseller/partner information
- ResellerName (For partner analysis)

---


