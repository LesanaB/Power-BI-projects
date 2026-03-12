# Power BI – Dátový model a DAX Measures
## Zadanie: Dashboard pre čas dodania služby & úspešnosť objednávok

---

## 1. Prehľad dátového modelu (Star Schema)

```
┌─────────────────┐       ┌───────────────────────────┐       ┌──────────────────────┐
│   dim_Calendar   │       │       fact_Orders          │       │     dim_Teams        │
│─────────────────│       │───────────────────────────│       │──────────────────────│
│ Date        (PK)│◄──────│ CREATED_DT_Date       (FK)│       │ District_Town_Key(PK)│
│ Year            │       │ District_Town_Key     (FK)│──────►│ DISTRICT             │
│ MonthOfYear     │       │ ID                        │       │ TOWN                 │
│ MonthName       │       │ CREATED_DT                │       │ TEAM_NAME            │
│ QuarterOfYear   │       │ END_DT                    │       │ REGION               │
│ DayOfWeek       │       │ ORDER_STATUS_NAME_SK      │       └──────────────────────┘
│ WeekNumber      │       │ TECHNOLOGY                │
│ ...             │       │ DISTRICT                  │
└─────────────────┘       │ TOWN                      │
                          │ REALIZATOR                │
                          │ Duration_Days (calc)      │
                          │ IsSuccessful (calc)       │
                          └───────────────────────────┘
```

**Vzťahy (Relationships):**
- `dim_Calendar[Date]` → `fact_Orders[CREATED_DT_Date]` (1:N)
- `dim_Teams[District_Town_Key]` → `fact_Orders[District_Town_Key]` (1:N)

---

## 2. Power Query – Nastavenie tabuliek

### 2.1 Fact tabulka: Orders (hárok "Detail")

V Power Query editore po načítaní hárku "Detail":

```
// Applied Steps:
1. Premenovať query na "fact_Orders"
2. Zmeniť typ stĺpcov:
   - CREATED_DT → DateTime
   - END_DT → DateTime
3. Pridať vypočítaný stĺpec – trvanie v kalendárnych dňoch:
   = Table.AddColumn(#"Changed Type", "Duration_Days", 
       each Duration.TotalDays([END_DT] - [CREATED_DT]), type number)
4. Pridať stĺpec pre dátum (len dátumová časť CREATED_DT):
   = Table.AddColumn(#"Added Duration", "CREATED_DT_Date", 
       each DateTime.Date([CREATED_DT]), type date)
5. Pridať stĺpec IsSuccessful (boolean):
   = Table.AddColumn(#"Added Date", "IsSuccessful", 
       each [ORDER_STATUS_NAME_SK] = "Objednávka ukončená", type logical)
6. Pridať kompozitný kľúč pre join s dim_Teams:
   = Table.AddColumn(#"Added IsSuccessful", "District_Town_Key", 
       each [DISTRICT] & "-" & [TOWN], type text)
```

**Power Query M kód (celý):**
```m
let
    Source = Excel.Workbook(File.Contents("Data_orders.xlsx"), null, true),
    Detail_Sheet = Source{[Item="Detail",Kind="Sheet"]}[Data],
    PromotedHeaders = Table.PromoteHeaders(Detail_Sheet, [PromoteAllScalars=true]),
    ChangedType = Table.TransformColumnTypes(PromotedHeaders, {
        {"ID", Int64.Type}, {"CREATED_DT", type datetime}, {"END_DT", type datetime},
        {"ORDER_STATUS_NAME_SK", type text}, {"TECHNOLOGY", type text},
        {"DISTRICT", type text}, {"TOWN", type text}, {"REALIZATOR", type text}
    }),
    AddDuration = Table.AddColumn(ChangedType, "Duration_Days", 
        each Duration.TotalDays([END_DT] - [CREATED_DT]), type number),
    AddCreatedDate = Table.AddColumn(AddDuration, "CREATED_DT_Date", 
        each DateTime.Date([CREATED_DT]), type date),
    AddIsSuccessful = Table.AddColumn(AddCreatedDate, "IsSuccessful", 
        each [ORDER_STATUS_NAME_SK] = "Objednávka ukončená", type logical),
    AddKey = Table.AddColumn(AddIsSuccessful, "District_Town_Key", 
        each [DISTRICT] & "-" & [TOWN], type text)
in
    AddKey
```

### 2.2 Dimenzia: Teams (hárok "Prevodovnik")

```m
let
    Source = Excel.Workbook(File.Contents("Data_orders.xlsx"), null, true),
    Prevodovnik_Sheet = Source{[Item="Prevodovnik",Kind="Sheet"]}[Data],
    PromotedHeaders = Table.PromoteHeaders(Prevodovnik_Sheet, [PromoteAllScalars=true]),
    ChangedType = Table.TransformColumnTypes(PromotedHeaders, {
        {"DISTRICT", type text}, {"TOWN", type text},
        {"TEAM_NAME", type text}, {"REGION", type text}
    }),
    AddKey = Table.AddColumn(ChangedType, "District_Town_Key", 
        each [DISTRICT] & "-" & [TOWN], type text)
in
    AddKey
```

> **Premenovať query na `dim_Teams`**
> 
> Stĺpec `District_Town_Key` je unikátny (1953 hodnôt, 0 duplikátov) a slúži ako Primary Key.

### 2.3 Dimenzia: Calendar (kalendárna tabuľka)

Postup:
1. V Power BI → Home → New Source → Blank Query
2. Otvoriť Advanced Editor
3. Vložiť funkciu z dodaného .txt súboru (fnDateTable)
4. Vytvoriť novú Blank Query a v Advanced Editor zadať:

```m
let
    Source = fnDateTable(#date(2024, 1, 1), #date(2025, 12, 31), 1)
in
    Source
```

> **Premenovať query na `dim_Calendar`**
> 
> Dátumový rozsah 2024-01-01 až 2025-12-31 pokrýva všetky dáta (niektoré CREATED_DT sú z apríla 2024).

5. V Model view označiť tabuľku `dim_Calendar` ako **Date Table** (Table tools → Mark as date table → stĺpec `Date`)

---

## 3. Vzťahy (Relationships) – Model View

Nastaviť v Model View:

| From | To | Typ | Kardinalita |
|------|----|-----|-------------|
| `dim_Calendar[Date]` | `fact_Orders[CREATED_DT_Date]` | Active | One-to-Many |
| `dim_Teams[District_Town_Key]` | `fact_Orders[District_Town_Key]` | Active | One-to-Many |

> **Pozn.:** TOWN samotný má 36 duplikátov (napr. "Belá" v okresoch Nové Zámky aj Žilina), preto sa používa kompozitný kľúč `DISTRICT-TOWN`. Join je 100% – všetkých 40 257 objednávok sa napárovalo.

---

## 4. DAX Measures

### 4.1 Základné počty

```dax
// Celkový počet objednávok
Total Orders = COUNTROWS(fact_Orders)

// Úspešne ukončené objednávky
Successful Orders = 
    CALCULATE(
        COUNTROWS(fact_Orders),
        fact_Orders[ORDER_STATUS_NAME_SK] = "Objednávka ukončená"
    )

// Neúspešné (stornované) objednávky
Failed Orders = 
    CALCULATE(
        COUNTROWS(fact_Orders),
        fact_Orders[ORDER_STATUS_NAME_SK] = "Inštalácia ukončená neúspešne"
    )
```

### 4.2 Úspešnosť objednávok (%)

```dax
Success Rate % = 
    DIVIDE(
        [Successful Orders],
        [Total Orders],
        0
    )
```

> Formát: Percentage (0.0%)

### 4.3 Čas dodania služby – Percentil 80 (hlavný KPI)

```dax
// P80 Duration – všetky objednávky (v kalendárnych dňoch)
P80 Duration All = 
    PERCENTILE.INC(
        fact_Orders[Duration_Days],
        0.8
    )

// P80 Duration – len úspešné objednávky
P80 Duration Successful = 
    CALCULATE(
        PERCENTILE.INC(fact_Orders[Duration_Days], 0.8),
        fact_Orders[ORDER_STATUS_NAME_SK] = "Objednávka ukončená"
    )

// P80 Duration – len neúspešné objednávky
P80 Duration Failed = 
    CALCULATE(
        PERCENTILE.INC(fact_Orders[Duration_Days], 0.8),
        fact_Orders[ORDER_STATUS_NAME_SK] = "Inštalácia ukončená neúspešne"
    )
```

### 4.4 Doplnkové štatistiky trvania

```dax
// Medián trvania (P50)
Median Duration = 
    MEDIAN(fact_Orders[Duration_Days])

// Priemerné trvanie
Avg Duration = 
    AVERAGE(fact_Orders[Duration_Days])

// Maximálne trvanie
Max Duration = 
    MAX(fact_Orders[Duration_Days])
```

### 4.5 Month-over-Month trend (pre grafy)

```dax
// P80 za predchádzajúci mesiac (pre porovnanie)
P80 Duration Previous Month = 
    CALCULATE(
        [P80 Duration All],
        DATEADD(dim_Calendar[Date], -1, MONTH)
    )

// MoM zmena v dňoch
P80 MoM Change = 
    [P80 Duration All] - [P80 Duration Previous Month]

// MoM zmena v percentách
P80 MoM Change % = 
    DIVIDE(
        [P80 MoM Change],
        [P80 Duration Previous Month],
        0
    )
```

### 4.6 Podiel interných vs. externých realizátorov

```dax
Internal Orders = 
    CALCULATE(
        COUNTROWS(fact_Orders),
        fact_Orders[REALIZATOR] = "INT"
    )

External Orders = 
    CALCULATE(
        COUNTROWS(fact_Orders),
        fact_Orders[REALIZATOR] = "EXT"
    )

Internal Share % = 
    DIVIDE([Internal Orders], [Total Orders], 0)
```

### 4.7 Dynamic Title measure (voliteľné)

```dax
Selected Region = 
    IF(
        ISFILTERED(dim_Teams[REGION]),
        "Región: " & SELECTEDVALUE(dim_Teams[REGION], "Viacero"),
        "Všetky regióny"
    )
```

---

## 5. Odporúčaná štruktúra dashboardu

### Strana 1: Prehľad (Executive Summary)

| Vizuál | Measures / Polia | Účel |
|--------|-------------------|------|
| **KPI karty** (3x) | `P80 Duration All`, `Success Rate %`, `Total Orders` | Hlavné čísla na prvý pohľad |
| **Line chart** | Os X: `dim_Calendar[MonthInCalendar]`, Y: `P80 Duration All` | Trend P80 počas roka |
| **Clustered bar chart** | Os X: `dim_Teams[REGION]`, Y: `P80 Duration All` | Porovnanie regiónov |
| **Donut chart** | Values: `Successful Orders`, `Failed Orders` | Pomer úspešných / neúspešných |
| **Slicery** | `dim_Teams[REGION]`, `dim_Calendar[MonthName]`, `fact_Orders[TECHNOLOGY]`, `fact_Orders[REALIZATOR]` | Filtrovanie |

### Strana 2: Detail podľa tímov

| Vizuál | Measures / Polia | Účel |
|--------|-------------------|------|
| **Matrix tabuľka** | Riadky: `TEAM_NAME`, Stĺpce: `MonthInCalendar`, Hodnoty: `P80 Duration All` | Heatmapa P80 po tímoch a mesiacoch |
| **Bar chart** | Os: `TEAM_NAME`, Y: `P80 Duration All`, `Success Rate %` | Ranking tímov |
| **Line chart** | Os X: `MonthInCalendar`, Y: `P80 Duration All`, Legend: `REGION` | Trend po regiónoch |
| **Table** | `TEAM_NAME`, `Total Orders`, `Success Rate %`, `P80 Duration All`, `Avg Duration` | Detailný prehľad |

---

## 6. Conditional Formatting (tipy)

Pre **Matrix** vizuál s P80 Duration:
- Zelenú farbu pozadia pre hodnoty **< 7 dní**
- Žltú pre **7–12 dní**
- Červenú pre **> 12 dní**

```
Format → Cell elements → Background color → Rules:
  If value <= 7 then Green
  If value > 7 and <= 12 then Yellow  
  If value > 12 then Red
```

---

## 7. Dátové fakty z analýzy

Pre referenciu pri tvorbe dashboardu:

| Metrika | Hodnota |
|---------|---------|
| Celkový počet objednávok | 40 257 |
| Úspešné objednávky | 31 806 (79,0%) |
| Neúspešné objednávky | 8 451 (21,0%) |
| P80 Duration (celkovo) | ~10,5 dňa |
| P80 Duration (úspešné) | ~10,3 dňa |
| P80 Duration (neúspešné) | ~10,8 dňa |
| Medián trvania | ~5,9 dňa |
| Obdobie dát | Jan 2025 – Okt 2025 |
| Počet regiónov | 3 (Západ, Stred, Východ) |
| Počet tímov | 19 |
| Technológie | Optical, Metallic, SAT, Mobile, Optic |
| Realizátor | INT (59%), EXT (41%) |

---

## 8. Checklist pred prezentáciou

- [ ] Kalendárna tabuľka označená ako Date Table
- [ ] Vzťahy správne nastavené (1:N)
- [ ] Measures vo vlastnom priečinku ("_Measures" tabuľka)
- [ ] Formáty: P80 na 1 desatinné miesto, Success Rate ako %, počty bez desatinných miest
- [ ] Slicery synchronizované medzi stránkami
- [ ] Conditional formatting na Matrix
- [ ] Titulok dashboardu s dynamic measure `Selected Region`
- [ ] Testované s filtrom na konkrétny región/tím
