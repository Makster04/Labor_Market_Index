
# All Indicators

## 🧲 **Labor Market Tightness**

*Measures employer demand and labor force engagement*
**Frequency:** Monthly

| Indicator                                | Source              |
| ---------------------------------------- | ------------------- |
| Job Openings Rate                        | JOLTS (`JTSJOL`)    |
| Hires Rate                               | JOLTS (`JTSHIR`)    |
| Prime-Age Labor Force Participation Rate | BLS (`LNS11300060`) |

**Feature Engineering:**

* `OpeningsPerUnemployed = Job Openings / Unemployed`
* `OpeningsPerHire = Job Openings / Hires`
* `PrimeLFPR_Z = z-score(Prime-Age LFPR)`
* `ParticipationGap = Pre-pandemic Prime LFPR – Current`


---

## 😣 **Labor Market Distress**

*Signals slack, hardship, and rising involuntary unemployment — from early shocks to persistent labor force detachment.*
**Frequency:** Monthly

| Indicator                                           | Source                        | Notes                                                            |
| --------------------------------------------------- | ----------------------------- | ---------------------------------------------------------------- |
| Unemployment Rate (U-3)                             | BLS (`UNRATE`)                | Headline unemployment, captures active jobseekers                |
| Unemployment Rate (U-6)                             | BLS (`U6RATE`)                | Broader measure incl. discouraged and underemployed              |
| Unemployment Rate (U-2)                             | BLS (`U2RATE`)                | Job losers and temp job completers; early distress signal        |
| Initial Jobless Claims                              | FRED (`ICSA`)                 | Weekly flow of new unemployment claims                           |
| Continued Jobless Claims                            | FRED (`CCSA`)                 | Signals persistent unemployment conditions                       |
| Median Weeks Unemployed                             | BLS (`UEMPMED`)               | Duration of joblessness; hardship signal                         |
| Long-term Unemployed Share                          | Derived (UEMP27OV / UNEMPLOY) | Share of unemployed without work for 27+ weeks                   |
| Part-Time for Economic Reasons                      | BLS (`LNS12032194`)           | Involuntary part-time workers                                    |
| Not in Labor Force – Want a Job Now                 | FRED (`NILFWJN`)              | Broad group who want work but are not searching                  |
| Marginally Attached (Want Job Now, Recently Looked) | BLS (`LNU05026645`)           | Subset of NILFWJN; looked in past 12 months but not last 4 weeks |
| Insured Unemployment Rate                           | FRED (`IURSA`)                | Unemployment claims as % of covered workforce                    |

---

### 🧠 **Feature Engineering:**

* `U6_U3_Spread = U6RATE - U3RATE`
  *Captures hidden slack beyond headline unemployment*
* `Unemployed27Share = UEMP27OV / UNEMPLOY`
  *Signals duration-based labor hardship*
* `InvoluntaryPartTimeRate = LNS12032194 / Employed`
  *Quantifies underemployment and hours cutbacks*
* `NILFWJN_Ratio = NILFWJN / NotInLaborForce`
  *Captures general desire for work across non-participants*
* `MarginalAttachmentRate = LNU05026645 / NotInLaborForce`
  *Stricter labor market detachment metric*
* `MedianWeeks_Z = z-score(MedianWeeksUnemployed)`
* `U2_Z = z-score(U2RATE)`
* `DistressIndex = mean(z-scores of all indicators above)`

---

Let me know if you'd like help visualizing how these move during historical recessions or integrating them into a PCA-based distress index.
          |

---

## 🔁 **Labor Mobility & Confidence**

*Tracks voluntary movement, layoffs, and temp labor demand*
**Frequency:** Monthly

| Indicator                                      | Source                |
| ---------------------------------------------- | --------------------- |
| Quit Rate                                      | JOLTS (`JTSQUR`)      |
| Layoffs and Discharges                         | JOLTS (`JTSLDL`)      |
| Total Separations                              | JOLTS (`JTSTSL`)      |
| Temporary Help Services Employment             | CES (`CES6054880001`) |
| Effective Federal Funds Rate (contextual only) | FRED (`FEDFUNDS`)     |

**Feature Engineering:**

* `QuitsToLayoffs = Quits / Layoffs`
* `VoluntaryExitRatio = Quits / Separations`
* `LayoffShock = % MoM change in Layoffs`
* `TempHelpMoM = % change in TEMPHELPS`
* `SeparationFlow_Z = z-score(Total Separations)`
* `FedFundsShock = % Δ in FEDFUNDS (contextual only)`

---

## 💸 **Compensation & Participation Signals**

*Measures wage pressure, real earnings growth, work intensity, and participation in the labor market*
**Frequency:** Monthly

| Indicator                                | Source                    | Notes                                          |
| ---------------------------------------- | ------------------------- | ---------------------------------------------- |
| Avg Weekly Earnings – Total Private      | CES (`CES0500000030`)     | Nominal earnings across majority of workforce  |
| Median Hourly Wage Growth – 3MMA         | ATL Fed (`FRBATLWGT3MMA`) | Smoothed median wage change, outlier-resistant |
| Prime-Age Employment-to-Population Ratio | BLS (`LNS12300060`)       | Core employment engagement (ages 25–54)        |
| Prime-Age Labor Force Participation Rate | BLS (`LNS11300060`)       | Best signal of labor force engagement          |
| Avg Weekly Hours – Total Private         | CES (`AWHAETP`)           | Reflects changes in workload and hiring demand |

---

### 🧠 **Feature Engineering:**

* `AvgHourlyEarning_YoY = YoY % change in Avg Weekly Earnings`
  *Captures nominal wage growth*
* `RealWageGrowth = AvgHourlyEarning_YoY – CPI_YoY`
  *Measures purchasing power of wages*
* `MedianWage3MMA_Z = z-score(Median Wage 3MMA)`
  *Standardized short-term wage signal*
* `PrimeEPOP_Z = z-score(Prime-Age Employment-to-Population)`
  *Normalized employment engagement*
* `PrimeLFPR_Trend = Current LFPR – LFPR 12 months ago`
  *Longitudinal participation shift*
* `WeeklyHours_Trend = Change in Avg Weekly Hours over 12 months`
  *Captures work intensity trends*

## ❄️ **Structural Freeze Confirmers (New Category)**

*Quarterly indicators that confirm structural slowdowns and investment freezes*
**Frequency:** Quarterly

| Indicator                               | Source              |
| --------------------------------------- | ------------------- |
| Labor Productivity (Output per Hour)    | BLS (`PRS85006092`) |
| Private Nonresidential Fixed Investment | BEA (`PNFI`)        |

**Feature Engineering:**

* `ProductivityTrend = QoQ % Δ in Labor Productivity`
* `InvestmentTrend = % change in PNFI`

---

Would you like help now grouping these into an index construction pipeline — e.g., clustering, PCA, or forecasting with regime transitions?

---

# Create the Indicies

To construct each of these labor market sub-indices — **Labor Tightness**, **Labor Distress**, **Mobility & Confidence**, and **Compensation Pressure** — you’ll follow a consistent **4-step methodology** for each:

---

### 🔧 GENERAL STEPS TO BUILD EACH INDEX:

1. **Normalize Indicators**
   Use **Z-scores** or **percentage changes** to standardize across units (e.g., rates vs. counts).

2. **Align Directionality**
   Ensure all components move in the same direction with respect to tightness/distress/etc. (e.g., higher = tighter).

3. **Aggregate Indicators**
   Combine via:

   * Mean of Z-scores (simple, interpretable)
   * PCA (captures primary common signal)
   * Weighted sum (if domain knowledge suggests)

4. **Optional: Smooth or Transform**
   Apply rolling average (e.g., 3-month) or exponential smoothing for stability.

---

