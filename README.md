# 🏦 Loan Default Analysis — Power BI Dashboard
https://app.powerbi.com/links/Q1vQu0d9zF?ctid=51697115-1ecd-42b5-b509-2d62c3919f76&pbi_source=linkShare

![Power BI](https://img.shields.io/badge/Power%20BI-F2C811?style=for-the-badge&logo=powerbi&logoColor=black)
![SQL Server](https://img.shields.io/badge/SQL%20Server-CC2927?style=for-the-badge&logo=microsoftsqlserver&logoColor=white)
![DAX](https://img.shields.io/badge/DAX-0078D4?style=for-the-badge&logo=microsoft&logoColor=white)
![Excel](https://img.shields.io/badge/Excel-217346?style=for-the-badge&logo=microsoftexcel&logoColor=white)

> An end-to-end Power BI project analysing **255,347 loan records** to uncover default patterns, demographic risk profiles, and year-over-year financial trends — built on a full data pipeline from SQL Server to Power BI Service.

---

## 📌 Project Overview

This project simulates a real-world banking analytics workflow. Raw loan data was ingested into **SQL Server**, connected to **Power BI Service via Gateway and Dataflow**, transformed in **Power Query**, and analysed using advanced **DAX measures and calculated columns** across 3 report pages.

The goal was to answer critical business questions around:
- Who is defaulting on loans and why?
- How does income, credit score, age, and employment type affect loan behaviour?
- What are the year-over-year trends in loan amounts and defaults?

---

## 🗂️ Dataset

| Property | Details |
|---|---|
| **Source** | Excel file imported into SQL Server |
| **Rows** | 255,347 records |
| **Key Columns** | LoanID, LoanAmount, Income, Age, CreditScore, EmploymentType, MaritalStatus, Education, Default (TRUE/FALSE), Loan_Date |
| **Target Variable** | `Default` — TRUE if borrower failed to repay |

---

## 🏗️ Data Pipeline Architecture

```
Excel File
    ↓
SQL Server (Data Storage)
    ↓
Power BI Gateway (On-Premise Connection)
    ↓
Power BI Dataflow (Cloud ETL)
    ↓
Power BI Desktop (Transformation + Modelling)
    ↓
Power BI Service (Published Report)
```

---

## 🔧 Data Transformation (Power Query)

Steps performed in Power Query Editor:

- ✅ Inspected **Column Quality**, **Column Profile**, and **Column Distribution** for all columns
- ✅ Verified and corrected **data types** for each column
- ✅ Identified date format as `MM-DD-YY` — handled via DAX calculated column
- ✅ Removed nulls and validated row counts against source

---

## 📊 Report Pages

### Page 1 — Loan Default & Overview

> High-level loan performance metrics and default trends

| Visual | DAX Measure / Column | Purpose |
|---|---|---|
| Line Chart | `Loan Amount by Purpose` | Total loan amount excluding blanks |
| Line Chart | `Average Income by Employment Type` | Avg income with single filter context |
| Line Chart | `Default Rate by Employment Type` | % default rate per employment type |
| Line Chart | `Average Loan by Age Groups` | Avg loan amount across age segments |
| Line Chart | `Default Rate by Year` | Year-wise default rate trend |

**Calculated Column:**
```dax
Year = YEAR('Loan_default'[Loan_Date_DD_MM_YYYY].[Date])
```

```dax
Age Groups =
IF('Loan_default'[Age] <=19, "Teen",
    IF('Loan_default'[Age]<=39,"Adults",
        IF('Loan_default'[Age]<=59,"Middle Age Adults",
        "Senior Citizens")))
```

**Key Measures:**
```dax
Loan Amount by Purpose =
SUMX(
    FILTER('Loan_default', NOT(ISBLANK('Loan_default'[LoanAmount]))),
    'Loan_default'[LoanAmount]
)
```

```dax
Average Income by Employment Type =
CALCULATE(
    AVERAGE('Loan_default'[Income]),
    ALLEXCEPT('Loan_default','Loan_default'[EmploymentType])
)
```

```dax
Default Rate by Employment Type =
VAR totalrecords = COUNTROWS(ALL('Loan_default'))
VAR DefaultCases = COUNTROWS(FILTER('Loan_default','Loan_default'[Default]=TRUE()))
RETURN
CALCULATE(DIVIDE(DefaultCases, totalrecords), ALLEXCEPT('Loan_default','Loan_default'[EmploymentType])) * 100
```

```dax
Default Rate by Year =
VAR totalloans =
    CALCULATE(COUNTROWS('Loan_default'),
    ALLEXCEPT('Loan_default', Loan_default[Year]))
VAR default =
    CALCULATE(COUNTROWS(FILTER('Loan_default','Loan_default'[Default]=TRUE())),
    ALLEXCEPT('Loan_default','Loan_default'[Year]))
RETURN
DIVIDE(default, totalloans) * 100
```

---

### Page 2 — Applicant Demographics & Financial Profile

> Demographic segmentation and credit risk profiling

| Visual | DAX Measure / Column | Purpose |
|---|---|---|
| Line Chart | `Median by Credit Score Bins` | Median loan amount by credit category |
| Line Chart | `Average Loan Amt (High Credit)` | Avg loan for high credit score applicants |
| Pie Chart | `Total Loan (Credit Bins)` | Adult loan distribution by credit & marital status |
| Clustered Column | `Total Loan (Middle Age Adults)` | Loan breakdown by mortgage & dependents |
| Line Chart | `Loans by Education Type` | Count of loans by education level |

**Calculated Column:**
```dax
Credit Score Bins =
IF('Loan_default'[CreditScore]<=400, "Very Low",
    IF('Loan_default'[CreditScore]<=450, "Low",
        IF('Loan_default'[CreditScore]<=650, "Medium",
        "High")))
```

**Key Measures:**
```dax
Median by Credit Score Bins =
MEDIANX('Loan_default', 'Loan_default'[LoanAmount])
```

```dax
Average Loan Amt (High Credit) =
AVERAGEX(
    FILTER('Loan_default', 'Loan_default'[Credit Score Bins]="High"),
    'Loan_default'[LoanAmount]
)
```

```dax
Total Loan (Credit Bins) =
CALCULATE(
    SUM('Loan_default'[LoanAmount]),
    'Loan_default'[Age Groups]="Adults",
    ALLEXCEPT('Loan_default',
        Loan_default[Age],
        'Loan_default'[Age Groups],
        'Loan_default'[CreditScore],
        'Loan_default'[Credit Score Bins])
)
```

```dax
Loans by Education Type =
COUNTROWS(FILTER('Loan_default', NOT(ISBLANK('Loan_default'[LoanID]))))
```

---

### Page 3 — Financial Risk Metrics

> Year-over-year trends, YTD analysis, and income-based risk decomposition

| Visual | DAX Measure / Column | Purpose |
|---|---|---|
| Line Chart | `YOY Loan Amount Change` | Year-over-year % change in loan volume |
| Line Chart | `YOY Default Loans Change` | Year-over-year % change in defaults |
| Ribbon Chart | `YTD Loan Amount` | YTD loans by credit score & marital status |
| Decomposition Tree | Sum of LoanAmount | Explained by Income Bracket → Employment Type |

**Calculated Column:**
```dax
Income Bracket =
SWITCH(
    TRUE(),
    'Loan_default'[Income] < 30000, "Low Income",
    'Loan_default'[Income] >= 30000 && 'Loan_default'[Income] < 60000, "Medium Income",
    'Loan_default'[Income] >= 60000, "High Income"
)
```

**Key Measures:**
```dax
YOY Loan Amount Change =
DIVIDE(
    CALCULATE(SUM('Loan_default'[LoanAmount]),
        'Loan_default'[Year] = YEAR(MAX('Loan_default'[Loan_Date_DD_MM_YYYY]))) -
    CALCULATE(SUM('Loan_default'[LoanAmount]),
        'Loan_default'[Year] = YEAR(MAX('Loan_default'[Loan_Date_DD_MM_YYYY]))-1),
    CALCULATE(SUM('Loan_default'[LoanAmount]),
        'Loan_default'[Year] = YEAR(MAX('Loan_default'[Loan_Date_DD_MM_YYYY]))-1),
    0
)
```

```dax
YOY Default Loans Change =
DIVIDE(
    CALCULATE(COUNTROWS(FILTER('Loan_default','Loan_default'[Default]=TRUE())),
        'Loan_default'[Year] = YEAR(MAX('Loan_default'[Loan_Date_DD_MM_YYYY]))) -
    CALCULATE(COUNTROWS(FILTER('Loan_default','Loan_default'[Default]=TRUE())),
        'Loan_default'[Year] = YEAR(MAX('Loan_default'[Loan_Date_DD_MM_YYYY]))-1),
    CALCULATE(COUNTROWS(FILTER('Loan_default','Loan_default'[Default]=TRUE())),
        'Loan_default'[Year] = YEAR(MAX('Loan_default'[Loan_Date_DD_MM_YYYY]))-1),
    0
) * 100
```

```dax
YTD Loan Amount =
CALCULATE(
    SUM('Loan_default'[LoanAmount]),
    DATESYTD('Loan_default'[Loan_Date_DD_MM_YYYY].[Date]),
    ALLEXCEPT('Loan_default',
        'Loan_default'[Credit Score Bins],
        'Loan_default'[MaritalStatus])
)
```

---

## ✅ Data Validation Approach

Every single measure and calculated column was validated using **3 methods**:

| Method | How |
|---|---|
| **Table Visual** | Added table in Power BI with filters applied to verify DAX output row by row |
| **Power BI Filter Pane** | Applied "not blank" and category filters to cross-check aggregated values |
| **Excel Pivot Table** | Replicated each measure logic in Excel pivot to confirm totals match |

This 3-way validation ensures **100% accuracy** of all reported metrics.

---

## 🛠️ Tools & Technologies

| Tool | Usage |
|---|---|
| **SQL Server** | Raw data storage and import from Excel |
| **Power BI Gateway** | On-premise to cloud data connection |
| **Power BI Dataflow** | Cloud-based ETL and data preparation |
| **Power Query** | Column profiling, type checks, transformations |
| **DAX** | 12 measures + 4 calculated columns |
| **Power BI Service** | Report publishing and sharing |
| **Excel** | Data validation via Pivot Tables |

---

## 💡 Key Business Insights

- 📌 **Employment Type** is a strong predictor of default — self-employed borrowers show higher default rates
- 📌 **Credit Score** directly impacts loan amounts — High credit (>650) borrowers take significantly larger loans
- 📌 **Middle Age Adults (40–59)** hold the highest total loan volume in the dataset
- 📌 **YOY trends** reveal increasing loan amounts with a corresponding rise in default rates
- 📌 **Low Income bracket** combined with self-employment shows the highest decomposition of loan risk

---

## 📁 Repository Structure

```
loan-default-powerbi/
├── LoanDefault_Dashboard.pbix     ← Power BI report file
├── loan_data.xlsx                 ← Source dataset (Excel)
├── screenshots/
│   ├── page1_loan_overview.png
│   ├── page2_demographics.png
│   └── page3_risk_metrics.png
└── README.md
```

---

## 🚀 How to Run This Project

1. Clone this repository
2. Open `LoanDefault_Dashboard.pbix` in **Power BI Desktop**
3. If prompted, update the data source path to your local `loan_data.xlsx`
4. Click **Refresh** to reload data
5. Explore all 3 report pages

> **Note:** To replicate the full pipeline (SQL Server → Gateway → Dataflow), you will need a Power BI Pro licence and SQL Server instance.

---

## 📝 Resume Line to Use

> *"Built an end-to-end Power BI dashboard on 255,347 loan records — designed a full data pipeline from SQL Server through Power BI Gateway and Dataflow, authored 12 DAX measures and 4 calculated columns across 3 report pages, and validated every metric using Power BI table visuals and Excel pivot tables."*

---

## 👤 Author

**Your Name**
📧 your.email@gmail.com
🔗 [LinkedIn](https://linkedin.com/in/yourprofile)
🌐 [Portfolio](https://yourusername.github.io)
🐙 [GitHub](https://github.com/yourusername)
