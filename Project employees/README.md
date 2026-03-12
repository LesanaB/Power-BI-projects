# DAX Measures – Overview

---

![Employees first tasks](https://github.com/user-attachments/assets/95f28eba-4743-4386-9bd7-d88f2c9b2580)

---

![Employees second tasks](https://github.com/user-attachments/assets/da21c471-1c22-4306-b81c-fdd0c0921d21)


## 📅 DATES AND EMPLOYMENT DURATION

### 1. Earliest Start
Converts the earliest start date from a number (YYYYMMDD) to an actual date. Dividing by 10000 extracts the year, MOD 100 extracts the month and day.
```dax
Earliest Start = 
DATE(
    INT(MIN(SQL_TEST_EMPLOYEE_HISTORY[START_DATE_KEY]) / 10000),
    MOD(INT(MIN(SQL_TEST_EMPLOYEE_HISTORY[START_DATE_KEY]) / 100), 100),
    MOD(MIN(SQL_TEST_EMPLOYEE_HISTORY[START_DATE_KEY]), 100)
)
```

### 2. odprac_mesiace
Total number of months worked. For each employee record, it calculates the difference between the start and end date in months and sums them all up.
```dax
odprac_mesiace = 
SUMX('SQL_TEST_EMPLOYEE_HISTORY',
    DATEDIFF('SQL_TEST_EMPLOYEE_HISTORY'[START_DATE_KEY_copy],
    'SQL_TEST_EMPLOYEE_HISTORY'[END_DATE_KEY_fix],MONTH))
```

### 3. odprac_mesiace_BA
Months worked filtered only for employees in Bratislava. Uses the odprac_mesiace measure and adds a city filter.
```dax
odprac_mesiace_BA = 
CALCULATE([odprac_mesiace], SQL_TEST_DEPARTMENTS[mesto] = "Bratislava")
```

### 4. pocet_odprac_dni
Total number of days worked. +1 is added so that the first day of employment is also counted.
```dax
pocet_odprac_dni = 
SUMX('SQL_TEST_EMPLOYEE_HISTORY',
    DATEDIFF('SQL_TEST_EMPLOYEE_HISTORY'[START_DATE_KEY_copy],
    'SQL_TEST_EMPLOYEE_HISTORY'[END_DATE_KEY_fix],DAY)+1)
```

### 5. pocet_odprac_rokov
Total number of years worked. Same logic as months and days, just with the YEAR parameter.
```dax
pocet_odprac_rokov = 
SUMX('SQL_TEST_EMPLOYEE_HISTORY',
    DATEDIFF('SQL_TEST_EMPLOYEE_HISTORY'[START_DATE_KEY_copy],
    'SQL_TEST_EMPLOYEE_HISTORY'[END_DATE_KEY_fix],YEAR))
```

---

## 📊 EMPLOYMENT DURATION AVERAGES

### 6. Priemer_clovek
Average number of months worked per employee. For each unique name, it calculates odprac_mesiace and then averages them. Warning: if two employees share the same name, they will be merged.
```dax
Priemer_clovek = 
AVERAGEX(VALUES(SQL_TEST_EMPLOYEE_HISTORY[EMPLOYEE_NAME]),[odprac_mesiace])
```

### 7. Priemerna dlzka pomeru v meste (mesiace)
Average employment duration per employee in months. When used in a visual with city on the axis, it automatically calculates the average for each city through the relationship with the departments table.
```dax
Priemerna dlzka pomeru v meste (mesiace) = 
AVERAGEX(
    VALUES(SQL_TEST_EMPLOYEE_HISTORY[UNIF_EMPLOYEE_MASTER_KEY]),
    [odprac_mesiace]
)
```

---

## 💰 SALARIES

### 8. MAXmesiac
The last (most recent) month in the salary table. A helper measure used by other measures.
```dax
MAXmesiac = 
MAX('SQL_TEST_SALARY_HISTORY (2)'[MONTH_KEY])
```

### 9. Aktualna vyplata
Current salary = salary in the last month. FILTER selects only rows where MONTH_KEY equals the last month.
```dax
Aktualna vyplata = 
CALCULATE(
    SUM('SQL_TEST_SALARY_HISTORY (2)'[SALARY]),
    FILTER(
        'SQL_TEST_SALARY_HISTORY (2)',
        'SQL_TEST_SALARY_HISTORY (2)'[MONTH_KEY] = [MAXmesiac]
    )
)
```

### 10. Priemerna mesacna mzda
Average monthly salary = total salary / months worked. DIVIDE protects against division by zero — if odprac_mesiace = 0, it returns 0.
```dax
Priemerna mesacna mzda = 
DIVIDE(
    SUM('SQL_TEST_SALARY_HISTORY (2)'[SALARY]),
    [odprac_mesiace],
    0
)
```

### 11. Priemer mzda na zamestnanca
Average of average monthly salaries across employees. Unlike Priemerna mesacna mzda, this gives correct subtotals and totals in visuals (e.g., average per city).
```dax
Priemer mzda na zamestnanca = 
AVERAGEX(
    VALUES(SQL_TEST_EMPLOYEE_HISTORY[UNIF_EMPLOYEE_MASTER_KEY]),
    [Priemerna mesacna mzda]
)
```

### 12. Priemer mzda na zamestnanca nova
DUPLICATE — same code as "Priemer mzda na zamestnanca". Recommended to delete one.
```dax
Priemer mzda na zamestnanca nova = 
AVERAGEX(
    VALUES(SQL_TEST_EMPLOYEE_HISTORY[UNIF_EMPLOYEE_MASTER_KEY]),
    [Priemerna mesacna mzda]
)
```

### 13. suma
Total sum of salaries grouped by employee name. GROUPBY is unnecessary here — Power BI groups automatically through the visual. Can be replaced with SUM('SQL_TEST_SALARY_HISTORY (2)'[SALARY]).
```dax
suma = 
CALCULATE(SUM('SQL_TEST_SALARY_HISTORY (2)'[SALARY]), 
    GROUPBY('UNIF emloyee salary ciselnik', 'UNIF emloyee salary ciselnik'[Mena zamestnancov]))
```

### 14. Measure
Total sum of salaries grouped by employee ID. Same as suma — GROUPBY is unnecessary, SUM(SALARY) is sufficient.
```dax
Measure = 
CALCULATE(SUM('SQL_TEST_SALARY_HISTORY (2)'[SALARY]),
    GROUPBY('UNIF emloyee salary ciselnik','UNIF emloyee salary ciselnik'[ID zamestnanca mzdy]))
```

### 15. percentil_63
63rd percentile of all salaries. ALL ignores all filters from the visual, so it always calculates from the entire table — the percentile stays constant regardless of the selected employee.
```dax
percentil_63 = 
CALCULATE(PERCENTILE.INC('SQL_TEST_SALARY_HISTORY (2)'[SALARY],0.63),
    ALL(SQL_TEST_EMPLOYEE_HISTORY))
```

---

## 👥 EMPLOYEE COUNTS

### 16. Pocet zamestnancov
Number of unique employees by UNIF key. DISTINCTCOUNT counts only unique values, so even if an employee has 10 rows, they are counted only once.
```dax
Pocet zamestnancov = 
DISTINCTCOUNT(SQL_TEST_EMPLOYEE_HISTORY[UNIF_EMPLOYEE_MASTER_KEY])
```

### 17. pocet_zamestnancov
Total number of rows in the table. WARNING: this is NOT the number of employees — one employee has multiple rows (history). For headcount use DISTINCTCOUNT instead.
```dax
pocet_zamestnancov = 
COUNTROWS(SQL_TEST_EMPLOYEE_HISTORY)
```

### 18. active sales
Number of active salespersons — only those where LAST_FLAG = "Y". DISTINCTCOUNT ensures each employee is counted only once.
```dax
active sales = 
CALCULATE(DISTINCTCOUNT(SQL_TEST_EMPLOYEE_HISTORY[UNIF_EMPLOYEE_MASTER_KEY]),
    SQL_TEST_EMPLOYEE_HISTORY[LAST_FLAG] = "Y")
```

---

## 🏆 IDENTIFICATION

### 19. Najstarší predajca
Finds the name of the employee with the earliest start date (longest tenure). First, MINX finds the smallest START_DATE_KEY, then CALCULATE filters rows with that date and returns the name. MIN(EMPLOYEE_NAME) is used only as an aggregation function — CALCULATE requires aggregation.
```dax
Najstarší predajca = 
VAR EarliestDate = 
    MINX(
        SQL_TEST_EMPLOYEE_HISTORY,
        SQL_TEST_EMPLOYEE_HISTORY[START_DATE_KEY]
    )
RETURN
    CALCULATE(
        MIN(SQL_TEST_EMPLOYEE_HISTORY[EMPLOYEE_NAME]),
        SQL_TEST_EMPLOYEE_HISTORY[START_DATE_KEY] = EarliestDate
    )
```

---


### 20. groupby_meno
```dax
groupby_meno = 
SUMX(SQL_TEST_EMPLOYEE_HISTORY,
    GROUPBY(SQL_TEST_EMPLOYEE_HISTORY,SQL_TEST_EMPLOYEE_HISTORY[EMPLOYEE_NAME]))
```
