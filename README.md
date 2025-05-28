# Measuring How Frozen & Imbalanced the Job Market Is

### *(01-03-2006 to 03-01-2025)*

## Project Overview

This project constructs and analyzes a set of **labor market health metrics** by combining publicly available economic indicators to assess how "frozen" or "imbalanced" the U.S. job market is over time. It blends demand and supply-side labor data to engineer composite indicators and visualize market trends.

---

## Key Features

### Derived Supply & Demand Metrics

| **Metric**                | **Formula**                              | **Insight**                          | **Type** |
| ------------------------- | ---------------------------------------- | ------------------------------------ | -------- |
| OpeningsPerUnemployed     | Job Openings Rate / Unemployment Level   | Signals labor tightness              | Demand   |
| OpeningsPerHire           | Job Openings Rate / Hires Rate           | Hiring friction or mismatch          | Demand   |
| HiresPerUnemployed        | Hires Rate / Unemployment Level          | Efficiency of absorbing unemployed   | Demand   |
| QuitsPerUnemployed        | Quits Rate / Unemployment Level          | Worker confidence                    | Supply   |
| LayoffsPerOpening         | Layoffs / Job Openings Rate              | Contradictory signals                | Supply   |
| QuitsPerLayoffs           | Quits Rate / Layoffs                     | Voluntary vs involuntary separations | Supply   |
| NILFWJNPerPop             | NILFWJN / Total Population               | Hidden slack in labor                | Supply   |
| MarginallyAttachedPerNILF | Marginally Attached / Not in Labor Force | Re-entry potential                   | Supply   |
| CPIYOY                    | %Δ CPI over 12 months                    | Inflation trend                      | Demand   |
| TempHelpEmploymentYoY     | %Δ Temp Jobs YoY                         | Leading demand signal                | Demand   |
| AvgWeeklyEarningYoY       | %Δ Avg Weekly Earnings YoY               | Wage growth                          | Demand   |
| RealAvgWeeklyEarningsYoY  | Earnings YoY - CPIYOY                    | Real wage pressure                   | Supply   |
| U6\_U3\_Spread            | U6 - U3 Unemployment Rate                | Underemployment spread               | Supply   |
| HourlyEarningEstimate     | Weekly Earnings / Weekly Hours           | Adjusted wage pressure               | Demand   |
| ApplicantsPerJobProxy     | Unemployment Level / Job Openings Rate   | Job seeker congestion                | Supply   |

---

## Data Sources & Loading

The project uses a large set of CSV files sourced from labor market datasets including:

* Layoffs, Hires, Job Openings, Quits
* CPI, Weekly Earnings & Hours
* Unemployment Measures (U2, U3, U6)
* Labor Force & Population Data
* Temp Help Employment, Wage Growth, Marginal Attachment

These datasets are read using `pandas` and merged into a unified SQL-based in-memory database via `sqlite3`.

---

## Methods

* Clean and normalize datasets
* Merge them on observation dates
* Derive meaningful ratios and indicators
* Calculate YoY percent changes and real wage growth
* Export a master `Supply_Demand_Indicators_df` for analysis and visualization

---

## Output

The merged and engineered dataset can be used for:

* Time series analysis of labor dynamics
* Composite index creation (e.g., Frozen Market Index)
* Tableau dashboards for visualization
* Policy research on hiring frictions, worker confidence, and labor slack

---

## Next Steps

* Build a time series forecasting model to determie the fure
* See how it contributes to economic indicators not included in calculations or how other indicators contribute to it (Ex. Inflation Rate, Interest Rates, Industrial Production, etc.)
* Evaluate index stability and predictive power for recessions or booms
