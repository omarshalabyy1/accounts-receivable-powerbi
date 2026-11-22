<p align="center"><img src="docs/header.svg" width="100%" alt="Accounts Receivable Power BI: 4 analysis pages, 14 measures, 8 tables in a star schema"></p>

<p align="center">
<img src="https://img.shields.io/badge/Power_BI-F2C811?style=for-the-badge&logo=powerbi&logoColor=black" alt="Power BI">
<img src="https://img.shields.io/badge/DAX-0E1630?style=for-the-badge" alt="DAX">
<img src="https://img.shields.io/badge/Power_Query-2563EB?style=for-the-badge" alt="Power Query">
<img src="https://img.shields.io/badge/TMDL_+_PBIR-1E293B?style=for-the-badge" alt="TMDL and PBIR project">
</p>

### Billed, collected and still owed, by client, project and age, in one report.

## The problem

Invoices, credit notes and receipts sit in one flat Excel sheet. Every week finance asks the same things: who owes us, how late is it, and which managing director or department is responsible.

## 🛠️ The solution

1. **Power Query** reads the sheet once, sets the types and works out the payment days.
2. **A star schema** splits the sheet into one fact table and five dimensions: Date, Client, Project (department, MD, project), Aging bands and Billing Timing.
3. **14 DAX measures** in one `_Measures` table, in four folders, each with a description.
4. **4 report pages** with a drawn background, a sidebar of navigation buttons and one question each.

## 📈 The result

| What | Count |
|---|---|
| Report pages (plus a home page) | 4 |
| Visuals, including buttons and text boxes | 85 |
| Tables in the model | 8 (fact, 5 dimensions, a measures table, a field parameter) |
| Relationships | 8, one filter direction, 3 of them inactive (extra dates) |
| Measures | 14, every one with a description |
| Model check (`pbip_check.py`) | 0 errors, 0 warnings |

| Page | Question it answers | Visuals | Measures used |
|---|---|---|---|
| Home | Where do I go? Four tiles open the four analysis pages. | 4 | none |
| AR Overview | How much is billed, collected and still outstanding? | 22 | 5 |
| Invoice and Payment Analysis | How do invoices turn into cash, and how long do they stay open? | 19 | 5 |
| Client Analysis | How does each client bill, pay and age? | 19 | 5 |
| Project MD&Department | Which MDs and departments bill and collect best? | 21 | 10 |

## 🏗️ For engineers

How the data moves, from the sheet to the pages. The boxes and counts are read from the project files.

![Data flow of the Accounts Receivable Power BI project: the Excel sheet Data is read by one Power Query query; the data model has a fact table, five dimension tables built from it, a Metric field parameter and 14 measures with 8 relationships; the measures feed 5 report pages.](docs/data-flow.drawio.svg)

### Data model

<!-- ![Data model: the fact table in the middle, five dimension tables around it](docs/data-model.png) -->

| Table | Role | Built how |
|---|---|---|
| `Entity Name (Data)` | Fact | Power Query from the Excel sheet; 5 calculated columns (Overdue Days, Aging Bucket, Billing Delay Days, Billing Timing, Project Key) |
| `Date` | Dimension, marked as the date table | Calculated: `CALENDAR` over the four date columns of the fact |
| `Client` | Dimension | Calculated: distinct client names |
| `Project` | Dimension with a department > MD > project hierarchy | Calculated: distinct rows on the hidden `Project Key` |
| `Aging` | Dimension | `DATATABLE` of 8 ageing bands with a sort column |
| `Billing Timing` | Dimension | `DATATABLE` of 5 values (Early, On Time, Delay, No Invoice, Unknown) |
| `Metric` | Field parameter | Switches the charts between collected and outstanding amounts |
| `_Measures` | Measures only | 14 measures |

Relationships run from each dimension to the fact in one direction. `Date` has four: Invoice Date is active; Estimated Bill Date, Credit Note Date and Receipt Date are inactive and used only through `USERELATIONSHIP` in measures.

### DAX measures

After the model come the measures. All of them sit in `_Measures`, grouped by topic.

#### Billing

**Total Billed Amount**: Invoiced amount including VAT/GST, before credit notes.

```dax
Total Billed Amount =
SUM ( 'Entity Name (Data)'[Billed Amount] )
```

**Total Credit Note Amount**: Credit notes issued, including VAT/GST.

```dax
Total Credit Note Amount =
SUM ( 'Entity Name (Data)'[Credit Note Amount] )
```

**Total Actual Billing Amount**: Billed amount net of credit notes.

```dax
Total Actual Billing Amount =
[Total Billed Amount] - [Total Credit Note Amount]
```

**Total Planned Billing Amount**: Billing planned for the estimated bill date.

```dax
Total Planned Billing Amount =
SUM ( 'Entity Name (Data)'[Planned Billing Amount] )
```

**Invoice Count**: Number of distinct invoices in the current filter context.

```dax
Invoice Count =
DISTINCTCOUNTNOBLANK ( 'Entity Name (Data)'[Invoice Number] )
```

#### Collections

**Total Collected Amount**: Cash received against invoices.

```dax
Total Collected Amount =
SUM ( 'Entity Name (Data)'[Collected Amount] )
```

**Total Outstanding Amount**: Amount billed and not yet collected.

```dax
Total Outstanding Amount =
SUM ( 'Entity Name (Data)'[Outstanding Amount] )
```

**Collection Efficiency**: Collected amount as a share of actual billing.

```dax
Collection Efficiency =
DIVIDE ( [Total Collected Amount], [Total Actual Billing Amount] )
```

**Collected by Receipt Date**: Cash by the date it was received (uses the inactive Receipt Date relationship); the other measures follow the invoice date.

```dax
Collected by Receipt Date =
CALCULATE ( [Total Collected Amount], USERELATIONSHIP ( 'Entity Name (Data)'[Receipt Date], 'Date'[Date] ) )
```

**Credit Notes by Credit Note Date**: Credit notes by the date they were issued (uses the inactive Credit Note Date relationship).

```dax
Credit Notes by Credit Note Date =
CALCULATE ( [Total Credit Note Amount], USERELATIONSHIP ( 'Entity Name (Data)'[Credit Note Date], 'Date'[Date] ) )
```

#### Ranking

**MD Billing Rank**: Rank of each managing director by billed amount (1 = highest).

```dax
MD Billing Rank =
RANKX ( ALL ( Project[Project MD] ), [Total Billed Amount], , DESC )
```

**MD Outstanding Rank**: Rank of each managing director by outstanding amount (1 = highest).

```dax
MD Outstanding Rank =
RANKX ( ALL ( Project[Project MD] ), [Total Outstanding Amount], , DESC )
```

#### Counts

**Total Projects**: Number of distinct projects in the current filter context.

```dax
Total Projects =
DISTINCTCOUNTNOBLANK ( 'Entity Name (Data)'[Project Key] )
```

**Number of MDs**: Number of distinct managing directors in the current filter context.

```dax
Number of MDs =
DISTINCTCOUNT ( 'Entity Name (Data)'[Project MD] )
```

## Open it

1. Open `Power BI Project #3-Accounts Receivable Analysis-restyled.pbip` in Power BI Desktop (PBIP with TMDL and PBIR enabled).
2. The query reads an Excel workbook that is not in this repo. In **Transform data**, change the file path in the `Entity Name (Data)` query to your copy of the sheet `Data`, then refresh.

## Data

One Excel workbook, sheet `Data`: invoice rows with client, project, planned and billed amounts, credit notes and receipts, and their dates. It is not included.
