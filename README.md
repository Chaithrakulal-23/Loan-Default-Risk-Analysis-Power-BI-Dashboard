# Loan Default Analysis - Power BI Dashboard

### Dashboard Link : https://app.powerbi.com/links/Q1vQu0d9zF?ctid=51697115-1ecd-42b5-b509-2d62c3919f76&pbi_source=linkShare

## Problem Statement

This dashboard helps banking institutions analyse loan default behaviour across their borrower portfolio. It enables risk teams to identify which customer segments are most likely to default, based on factors such as employment type, credit score, income bracket, and age group. Through year-over-year trend analysis, lenders can track whether default rates are improving or worsening over time, and take proactive measures accordingly.

Since self-employed borrowers show the highest default rates and low-income segments carry the greatest loan risk, institutions must focus their risk mitigation strategies on these groups. The dashboard also reveals that loan amounts are rising year-over-year alongside default rates — indicating a growing risk exposure that requires immediate attention.

---

## Dataset

| Property | Details |
|---|---|
| **Source** | Excel file imported into SQL Server |
| **Rows** | 255,347 records |
| **Key Columns** | LoanID, LoanAmount, Income, Age, CreditScore, EmploymentType, MaritalStatus, Education, Default (TRUE/FALSE), Loan_Date |
| **Target Variable** | `Default` — TRUE if borrower failed to repay |

---

## Data Pipeline Architecture

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

### Gateway Connection (Power BI Service)

![Gateway](screenshots/gateway.png)

---

## Steps Followed

- **Step 1** : Loaded raw loan data (Excel) into **SQL Server** as the primary data store to simulate a production banking environment.

- **Step 2** : Connected SQL Server to **Power BI Service** via **Personal Gateway** — established the on-premise to cloud data connection.

- **Step 3** : Created a **Power BI Dataflow** in the service for cloud-based ETL — linked the gateway connection to the dataflow for data preparation.

- **Step 4** : Opened **Power Query Editor** → under the **View** tab, enabled **Column Quality**, **Column Distribution**, and **Column Profile** for all columns to inspect data health.

![Power Query Profiling](screenshots/powerquery_profiling.png)

- **Step 5** : Verified and corrected **data types** for every column — ensured numeric, text, and date fields were correctly classified.

![Data Types](screenshots/powerquery_datatypes.png)

- **Step 6** : Identified the **Loan Date** column had mixed date formats (`MM/DD/YYYY` and `DD-MM-YYYY`). Resolved this using a **DAX calculated column** rather than Power Query to handle both formats cleanly.

- **Step 7** : Removed null values and validated final row count (255,347) against the source SQL Server table.

- **Step 8** : Built **Page 1 — Loan Default & Overview** with 5 visuals covering loan amounts by purpose, income by employment type, default rates, age group analysis, and year-wise default trends.

- **Step 9** : Created the following **Calculated Columns** for Page 1:

    **Year Column** — extracted year from the date field for YOY analysis:
    ```dax
    Year = YEAR('Loan_default'[Loan_Date_DD_MM_YYYY].[Date])
    ```

    **Age Groups Column** — segmented borrowers into age brackets:
    ```dax
    Age Groups =
    IF('Loan_default'[Age] <=19, "Teen",
        IF('Loan_default'[Age]<=39, "Adults",
            IF('Loan_default'[Age]<=59, "Middle Age Adults",
            "Senior Citizens")))
    ```

- **Step 10** : Created the following **DAX Measures** for Page 1:

    Loan Amount by Purpose (excludes blanks):
    ```dax
    Loan Amount by Purpose =
    SUMX(
        FILTER('Loan_default', NOT(ISBLANK('Loan_default'[LoanAmount]))),
        'Loan_default'[LoanAmount]
    )
    ```

    Average Income by Employment Type:
    ```dax
    Average Income by Employment Type =
    CALCULATE(
        AVERAGE('Loan_default'[Income]),
        ALLEXCEPT('Loan_default','Loan_default'[EmploymentType])
    )
    ```

    Default Rate by Employment Type:
    ```dax
    Default Rate by Employment Type =
    VAR totalrecords = COUNTROWS(ALL('Loan_default'))
    VAR DefaultCases = COUNTROWS(FILTER('Loan_default','Loan_default'[Default]=TRUE()))
    RETURN
    CALCULATE(DIVIDE(DefaultCases, totalrecords), ALLEXCEPT('Loan_default','Loan_default'[EmploymentType])) * 100
    ```

    Default Rate by Year:
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

    Snap of DAX measure in formula bar:

    ![DAX Measure](screenshots/dax_measure.png)

- **Step 11** : Built **Page 2 — Applicant Demographics & Financial Profile** with 5 visuals covering credit score segmentation, marital status distribution, education level, and age group loan breakdown.

- **Step 12** : Created the following **Calculated Column** for Page 2:

    Credit Score Bins — categorised credit scores into risk tiers:
    ```dax
    Credit Score Bins =
    IF('Loan_default'[CreditScore]<=400, "Very Low",
        IF('Loan_default'[CreditScore]<=450, "Low",
            IF('Loan_default'[CreditScore]<=650, "Medium",
            "High")))
    ```

- **Step 13** : Created the following **DAX Measures** for Page 2:

    Median Loan by Credit Score Bins:
    ```dax
    Median by Credit Score Bins =
    MEDIANX('Loan_default', 'Loan_default'[LoanAmount])
    ```

    Average Loan for High Credit Applicants:
    ```dax
    Average Loan Amt (High Credit) =
    AVERAGEX(
        FILTER('Loan_default', 'Loan_default'[Credit Score Bins]="High"),
        'Loan_default'[LoanAmount]
    )
    ```

    Total Loan by Credit Bins (Adults only):
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

    Loans by Education Type:
    ```dax
    Loans by Education Type =
    COUNTROWS(FILTER('Loan_default', NOT(ISBLANK('Loan_default'[LoanID]))))
    ```

- **Step 14** : Built **Page 3 — Financial Risk Metrics** with YOY trend charts, YTD ribbon chart, and a decomposition tree breaking down loan amount by income bracket and employment type.

- **Step 15** : Created the following **Calculated Column** for Page 3:

    Income Bracket — segmented borrowers by income level:
    ```dax
    Income Bracket =
    SWITCH(
        TRUE(),
        'Loan_default'[Income] < 30000, "Low Income",
        'Loan_default'[Income] >= 30000 && 'Loan_default'[Income] < 60000, "Medium Income",
        'Loan_default'[Income] >= 60000, "High Income"
    )
    ```

- **Step 16** : Created the following **DAX Measures** for Page 3:

    YOY Loan Amount Change:
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

    YOY Default Loans Change:
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

    YTD Loan Amount:
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

- **Step 17** : Validated every measure and calculated column using **3 independent methods**:
    - **Power BI Table Visual** — added table with filters to verify DAX output row by row
    - **Power BI Filter Pane** — applied category and "not blank" filters to cross-check aggregates
    - **Excel Pivot Table** — replicated each measure in Excel pivot to confirm totals match

    Snap of Excel validation:

    ![Excel Validation](screenshots/excel_validation.png)

- **Step 18** : Published the report to **Power BI Service**.

![Publish Success](screenshots/publish.png)

---

## Snapshot of Dashboard (Power BI Service)

### Page 1 — Loan Default & Overview
![Page 1 Power BI Service](screenshots/pagep1.png)

### Page 2 — Applicant Demographics & Financial Profile
![Page 2 Power BI Service](screenshots/page2p.png)

### Page 3 — Financial Risk Metrics
![Page 3 Power BI Service](screenshots/pagep3.png)

---

## Report Snapshot (Power BI Desktop)

### Page 1 — Loan Default & Overview
![Page 1 Desktop](screenshots/page1.png)

### Page 2 — Applicant Demographics & Financial Profile
![Page 2 Desktop](screenshots/page2.png)

### Page 3 — Financial Risk Metrics
![Page 3 Desktop](screenshots/page3.png)

---

## Insights

A 3-page report was created on Power BI Desktop and published to Power BI Service.

Following inferences can be drawn from the dashboard:

### [1] Default Rate by Employment Type

   Default Rate — Unemployed = 3.39%

   Default Rate — Part-time = 3.01%

   Default Rate — Self-employed = 2.86%

   Default Rate — Full-time = 2.36%

        Thus, unemployed and part-time borrowers carry the highest default risk.

### [2] Average Loan Amount by Age Group

    a) Adults (20–39)          - $127,901
    b) Middle Age Adults (40–59) - $127,459
    c) Senior Citizens (60+)   - $127,355
    d) Teens (≤19)             - $126,674

        Thus, Adults hold the highest average loan amount across all age segments.

### [3] Default Rate by Year

    a) 2013 - 11.62%
    b) 2014 - 11.50%
    c) 2015 - 11.70%
    d) 2016 - 11.75%
    e) 2017 - 11.50%
    f) 2018 - 11.60%

        Default rates peaked in 2016 at 11.75% and have shown slight improvement since,
        though they remain consistently above 11.5% — indicating a persistent systemic risk.

### [4] Loan Amount by Purpose

    a) Home     - 6,545M
    b) Business - 6,522M
    c) Education - 6,511M
    d) Auto     - 6,501M
    e) Other    - 6,498M

        Home loans account for the highest total loan amount across all purposes.

### [5] Some Other Insights

### Credit Score Impact

5.1) Median loan amount drops from **$128,397 (Low credit)** to **$127,149 (High credit)** — counterintuitively, lower credit score borrowers take larger loans.

5.2) High credit applicants (>650) show significantly better repayment behaviour despite similar loan sizes.

### Income Bracket & Employment (Decomposition Tree)

6.1) High Income borrowers account for **$21,731,557,581** in total loan amount.

6.2) Medium Income borrowers account for **$7,212,815,720**.

6.3) Low Income borrowers account for **$3,632,507,271**.

        Thus, the highest loan volume comes from High Income borrowers,
        but the highest default risk lies in the Low Income + Self-Employed segment.

### Education Type

7.1) Bachelor's degree holders — 64,366 loans (highest)

7.2) High School — 63,903 loans

7.3) Master's — 63,541 loans

7.4) PhD — 63,537 loans

        Loan count is fairly evenly distributed across education levels,
        suggesting education alone is not a strong differentiator of loan behaviour.
