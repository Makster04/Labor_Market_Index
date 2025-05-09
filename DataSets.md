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
| **U-6 Unemployment Rate** | BLS (`U6RATE`)      | Includes underemployed & discouraged workers |
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

## 🧊 COMPOSITE INDEX FEATURE ENGINEERING (Frozen Index)

To build the **Frozen Job Market Index**, follow this workflow:

### 1. **Standardize (Z-score)** the following variables:

| Direction        | Variable                                 |
| ---------------- | ---------------------------------------- |
| ⬆️ = More Frozen | Median Weeks Unemployed                  |
| ⬆️ = More Frozen | Unemployment Rate (U-3)                  |
| ⬆️ = More Frozen | Initial Jobless Claims                   |
| ⬆️ = More Frozen | U6–U3 Spread                             |
| ⬇️ = More Frozen | Beveridge Ratio / OpeningsPerUnemployed  |
| ⬇️ = More Frozen | Quit Rate                                |
| ⬇️ = More Frozen | Quits-to-Layoffs Ratio                   |
| ⬇️ = More Frozen | Prime-Age Labor Force Participation Rate |
| ⬇️ = More Frozen | Prime-Age Employment-to-Population Ratio |
| ⬇️ = More Frozen | Real Wage Growth                         |
| ⬇️ = More Frozen | Avg Weekly Hours – Total Private         |
| ⬇️ = More Frozen | Median Hourly Wage Growth – 3MMA         |

---

### 2. **Combine** into a composite score:

* Option 1: Equal-weight **average**
* Option 2: **PCA** to extract dominant “freeze” signal

---

### 3. **Optional Enhancements**:

* Apply a **3–6 month rolling mean** to smooth volatility
* Normalize final index on a **0–1 scale**
* Classify into tiers:

  * **Loose**
  * **Balanced**
  * **Mild Freeze**
  * **Severe Freeze**

---


Great question — now that you've finalized the **set of indicators for the Frozen Job Market Index**, you’re likely wondering:

> What should I do with all the **other indicators or engineered features** I calculated earlier but didn’t include in the final index?

---

## ✅ Here are 5 smart ways to use your **extra features** outside the Frozen Index:

---

### 1. 🧩 **Build Thematic Sub-Indices**

Use your remaining features to create **focused indices** that offer deeper views into specific labor market dimensions:

| Index Name                  | What to Include                                      | Use Case                               |
| --------------------------- | ---------------------------------------------------- | -------------------------------------- |
| `DistressIndex`             | Continued Claims, Long-term Unemployed %, U6 Rate    | Visualizing structural unemployment    |
| `CompensationPressureIndex` | Real Wage Growth, WeeklyHours\_Trend, MedianWage3MMA | Shows wage inflation and job intensity |
| `MobilityIndex`             | VoluntaryExitRatio, Total Separations, LayoffShock   | Measures worker confidence in quitting |

Use z-scores and `.mean(axis=1)` just like you did for the Frozen Index.

---

### 2. 📊 **Use for Contextual Dashboards or Heatmaps**

Even if a variable isn’t in the core index, it can **enrich your dashboard**:

* Show heatmaps of current vs. historical percentiles
* Display “Top 3 signals tightening” or “Cooling fastest”
* Track **leading indicators** ahead of turning points

---

### 3. 🧠 **Use as SHAP Features in a Classification Model**

Train a model to **predict freeze severity tiers** (`Loose`, `Balanced`, etc.), then use:

* All indicators as input features
* SHAP values to explain which variables **drive severity**

This adds interpretability and allows automated early warning.

---

### 4. ⏱️ **Backtesting & Scenario Analysis**

Use the extra indicators to **test how sensitive the Frozen Index is** to:

* Wage growth shifts
* Jobless claims surges
* Participation collapses

You can build scenario tools: *"What if RealWageGrowth drops 2%?"*

---

### 5. 📁 **Feature Library for Expansion**

Keep them in your dataset as a **feature library**:

* Ready for future index updates
* Available for new models (e.g. Recession Forecasting, Job Market Tightness Score)
* Valuable if segmenting by **industry**, **region**, or **demographic**

---

### ✅ Summary Table:

| Action                             | Benefit                                |
| ---------------------------------- | -------------------------------------- |
| Sub-indices (e.g. `DistressIndex`) | Deeper insights into market segments   |
| Dashboard visuals                  | Enhanced storytelling + trend tracking |
| Model features (e.g. SHAP)         | Interpretation and automation          |
| Scenario testing                   | What-if planning and policy tools      |
| Feature library                    | Future-proofing your labor analytics   |

---

Would you like help organizing the leftovers into grouped thematic sub-indices or deciding which ones to monitor on a live dashboard?


