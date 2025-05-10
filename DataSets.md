## 🔒 LABOR MARKET TIGHTNESS

**Indicators:**

| Indicator                                           | Source              | Notes                                                |
| --------------------------------------------------- | ------------------- | ---------------------------------------------------- |
| Job Openings Rate                                   | JOLTS (`JTSJOL`)    | Total Nonfarm or by industry                         |
| Hires Rate                                          | JOLTS (`JTSHIR`)    | Hiring flow                                          |
| Beveridge Ratio                                     | Derived             | = Job Openings / Unemployed                          |
| Openings per Hire                                   | Derived             | = Job Openings / Hires                               |
| **Prime-Age Labor Force Participation Rate (LFPR)** | BLS (`LNS11300060`) | Clean measure of active labor engagement (25–54 y/o) |

**Feature Engineering Ideas:**

* `OpeningsPerUnemployed = Job Openings / Unemployed`
* `OpeningsPerHire = Job Openings / Hires`
* `PrimeLFPR_Zscore = z-score(LNS11300060)`
* `ParticipationGap = Pre-pandemic Prime LFPR – Current Prime LFPR`
* Normalize using Z-scores or rolling averages

---

## 😣 LABOR MARKET DISTRESS

**Indicators:**

| Indicator                 | Source              | Notes                                        |
| ------------------------- | ------------------- | -------------------------------------------- |
| Unemployment Rate (U-3)   | BLS (`UNRATE`)      | Official unemployment rate                   |
| U-6 Unemployment Rate | BLS (`U6RATE`)      | Includes underemployed & discouraged workers |
| Initial Jobless Claims    | FRED (`ICSA`)       | Weekly job loss inflow                       |
| Continued Jobless Claims  | FRED (`CCSA`)       | Persistent unemployment inflow               |
| Median Weeks Unemployed   | BLS (`LNS13008275`) | Duration signal of unemployment hardship     |
| Long-term Unemployed (%)  | Derived             | = UEMP27OV / UNEMPLOY                        |

**Feature Engineering Ideas:**

* `Unemployed27Share = UEMP27OV / UNEMPLOY`
* `Lagged_Claims = Rolling average of ICSA over 4–8 weeks`
* `MedianWeeks_Zscore`
* `U6_U3_Spread = U6RATE - UNRATE`
* Combine into a `DistressIndex` using PCA or averaging

---

## 🔁 LABOR MOBILITY / CONFIDENCE

**Indicators:**

| Indicator              | Source           | Notes                                                                 |
| ---------------------- | ---------------- | --------------------------------------------------------------------- |
| Quit Rate              | JOLTS (`JTSQUR`) | Lower values suggest fear of switching jobs                           |
| Quits-to-Layoffs Ratio | Derived          | = Quits / Layoffs; higher = more confidence                           |
| Layoffs and Discharges | JOLTS (`JTSLDL`) | Indicator of involuntary exits                                        |
| **Total Separations**  | JOLTS (`JTSTSL`) | Total exits; useful for computing voluntary/involuntary flow dynamics |

**Feature Engineering Ideas:**

* `QuitsToLayoffs = Quits / Layoffs`
* `VoluntaryExitRatio = Quits / Total Separations`
* `LayoffShock = % change in Layoffs MoM`
* `SeparationFlow_Z = z-score(Total Separations)`
* Normalize using trend slopes or z-scores

---

## 💸 COMPENSATION & PARTICIPATION SIGNALS

**Indicators:**

| Indicator                                | Source                          | Notes                                                             |
| ---------------------------------------- | ------------------------------- | ----------------------------------------------------------------- |
| Avg Weekly Earnings – Total Private      | CES (`CES0500000030`)           | Nominal wages for majority of workforce                           |
| Median Hourly Wage Growth – 3MMA         | FRB ATL (`FRBATLWGT3MMAUMHWGO`) | Real-time, outlier-resistant wage pressure indicator              |
| Prime-Age Employment-to-Population Ratio | BLS (`LNS12300060`)             | Measures active employment engagement for 25–54 y/o               |
| Prime-Age Labor Force Participation Rate | BLS (`LNS11300060`)             | Best signal of core labor market participation                    |
| CPI Inflation (YoY % Change)             | FRED (`CPIAUCSL`)               | Used to deflate wages for purchasing power                        |
| Avg Weekly Hours – Total Private         | CES (`AWHAETP`)                 | Gauges workload; leading indicator for hours-based hiring freezes |

**Feature Engineering Ideas:**

* `AWE_YoY = AWE.pct_change(periods=12) * 100`
* `RealWageGrowth = AWE_YoY - CPI_YoY`
* `WageGrowth_YoY = AWE_YoY`
* `MedianWage3MMA_Z = z-score(FRBATL_Wage3MMA)`
* `PrimeAgeEPOP_Zscore`
* `PrimeLFPR_Trend = Current – 12-month lag`
* `WeeklyHours_Trend = Δ AWHAETP over 12 months`
* Include wage and hour trends in a `CompensationPressureIndex`

---

Yes, creating an **overall Frozen Job Market Index** by combining your four sub-indices absolutely makes sense—and it’s a powerful way to synthesize multiple labor dimensions into a single interpretable signal. You’ve already built the necessary building blocks.

---

### 🔄 **What the Frozen Market Index Represents**

It captures the *composite health or dysfunction* of the labor market by balancing:

* **Tightness** (demand strength),
* **Stress** (distress/insecurity),
* **Compensation Pressure** (wage/hours signals),
* **Mobility** (worker confidence in switching).

---

### ✅ Step-by-Step: Create the Composite Index

#### 1. **Normalize all sub-indices (Z-score or MinMax)**

If they’re already z-scored, you can average them directly. Otherwise:

```python
from sklearn.preprocessing import StandardScaler

index_vars = [
    'Labor_Tightness_Index',
    'Labor_Stress_Index',
    'Compensation_Pressure_Index',
    'Mobility_Voluntariness_Index'
]

scaler = StandardScaler()
LMT_df[index_vars] = scaler.fit_transform(LMT_df[index_vars])
```

#### 2. **Determine Directionality**

To make the **Frozen Index** intuitive (higher = more frozen), invert the scores that reflect *positive conditions*:

* **Tightness** and **Mobility** are *anti-freeze* indicators → multiply by -1.

```python
LMT_df['Frozen_Market_Index'] = (
    -LMT_df['Labor_Tightness_Index'] +   # inverted
     LMT_df['Labor_Stress_Index'] +      # already distress-based
     LMT_df['Compensation_Pressure_Index'] +  # wage pressure can be inflationary/freeze-like
    -LMT_df['Mobility_Voluntariness_Index']   # inverted
) / 4
```

> 🔁 **Optional**: Weight components differently (e.g., stress could have a higher weight during recessions)

---

### 📈 Interpretation

* **Higher values** = Job market is more "frozen" (hard to hire, low quits, high unemployment/claims).
* **Lower values** = Healthier, dynamic market.

---

### ✅ Optional Enhancements

* Apply **rolling mean** to smooth volatility:

```python
LMT_df['Frozen_Market_Index_Smoothed'] = LMT_df['Frozen_Market_Index'].rolling(3, center=True).mean()
```

* Plot over time with recession bands or hiring freezes for comparison.

---

Would you like help visualizing it, or building thresholds (e.g. “Loose”, “Mild Freeze”, “Severe Freeze”)?
