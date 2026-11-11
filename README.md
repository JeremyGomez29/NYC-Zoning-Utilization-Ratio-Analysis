# Zoning Utilization in New NYC Residential Buildings (2012–2025)

**Author:** Jeremy Gomez

This project looks at what makes developers in New York City build to the full **Floor Area Ratio (FAR)** their zoning allows. It covers every residential building that received a Certificate of Occupancy between July 2012 and November 2025, about 233,000 observations, and uses a logistic regression, bootstrapped confidence intervals and a time-series smoother to identify the economic, regulatory and physical factors behind "maxing out" a lot.

---

## Background

NYC's rental vacancy rate fell to 1.4% in 2025, and median Manhattan rent reached $4,750. Land in the city is among the most expensive anywhere, so a developer has a strong incentive to use all of a lot's buildable floor area, and the city gains new homes when they do. Each zoning district caps building size with a FAR, which is the building's floor area divided by the lot's area. Under a FAR of 1.0, for example, a developer could build one story covering the whole lot or two stories covering half of it.

This project measures the gap between the floor area zoning allows and what developers actually build, and asks what explains it.

## Research objectives

1. **Find the predictors of maximum utilization.** A logistic regression estimates how land value, borough, zoning category, lot shape, landmark status, lot size and job type affect the probability that a building is *Maxed Out*.
2. **Check robustness with bootstrapping.** Non-parametric bootstrapping gives 95% confidence intervals for the regression coefficients.
3. **Show how utilization changed over time.** A Generalized Additive Model (GAM) traces zoning utilization across three events: the 2016 Zoning for Quality and Affordability (ZQA) amendments, the COVID-19 shutdown and the 2022 expiration of the 421-a tax exemption.

> **Note:** The proposal planned a LOESS smoother. With more than 200k observations, LOESS wouldn't run locally or in a high-RAM Google Colab session, so a GAM (`s(x, bs = "cs")`) was used instead. It costs much less to compute and gives similar results.

## Key definitions

| Variable | Definition |
|---|---|
| `FAR_Utilization` | Built FAR ÷ Allowable (residential) FAR |
| `Maxed_Out` | `1` if `FAR_Utilization > 0.9`, otherwise `0` |
| `Land_Value_Per_SqFt` | Assessed land value ÷ lot area |

---

## Repository contents

| File | Description |
|---|---|
| `Project_Code.R` | Full analysis pipeline: data cleaning and joins, EDA, chi-squared tests, logistic regression, VIF, bootstrap and the GAM time-series plot |
| `Project_Proposal.pdf` | Original project proposal |
| `Project_Report.pdf` | Final written report with methods, results and policy recommendations |
| `Project_Presentation.pptx` / `.pdf` | Slide deck summarizing the project |

## Data

All four datasets come from [NYC Open Data](https://opendata.cityofnewyork.us/). They are **not included** in this repository because of their size. Download them as CSVs and save them under the filenames below:

| Dataset | Expected filename | Purpose |
|---|---|---|
| DOB Certificate of Occupancy (BIS, Jul 2012 – Mar 2021) | `DOB_Certificate_of_Occupancy_2012_2021.csv` | Certificates of Occupancy issued before the system migration |
| DOB NOW: Certificate of Occupancy (Mar 2021 – Nov 2025) | `DOB_NOW_Certificate_of_Occupancy.csv` | Certificates of Occupancy issued after the migration to DOB NOW |
| DOB Job Application Filings | `DOB_Job_Application_Filings.csv` | Proposed dwelling unit counts, used to filter out non-residential buildings |
| Primary Land Use Tax Lot Output (PLUTO) | `PLUTO.csv` | Built FAR, allowable FAR, lot size, assessed land value, zoning, landmark and lot-shape fields |

### Processing pipeline

1. The two Certificate of Occupancy datasets are cleaned and stacked.
2. A 10-digit **BBL** (Borough-Block-Lot) key is built for each record.
3. Missing dwelling unit counts are filled from DOB Job Application Filings, joined on job number.
4. PLUTO lot attributes are joined on BBL.
5. Rows are filtered to residential buildings (≥ 1 dwelling unit) with valid FAR, lot size and land value. New buildings with a built FAR below 0.05 are also dropped.
6. The final dataset has about **233k rows and 15 variables**.

---

## Methods

- **Exploratory analysis:** a Pearson correlation matrix of the continuous variables, chi-squared tests of each categorical predictor against `Maxed_Out`, and proportion bar charts.
- **Logistic regression:** `Maxed_Out ~ Borough + Zoning_Category + Is_Landmark + Is_Irregular + Job_Type + LotSize + Land_Value_Per_SqFt`. Lot frontage was left out because its correlation with lot size (r = 0.87) would introduce collinearity.
- **Multicollinearity check:** all VIF values are below 2.1.
- **Bootstrap:** non-parametric 95% confidence intervals for the coefficients.
- **Temporal analysis:** a GAM smoother of utilization against issue date, with reference lines for the three policy and market events.

## Key findings

- **Job type is the strongest positive predictor.** New buildings have about **2.9× the odds** of maxing out compared with alterations (coefficient 1.075).
- **Borough matters a lot.** Compared with the Bronx baseline, Brooklyn's odds of maxing out are higher (OR ≈ 1.55), while Queens (OR ≈ 0.39) and Staten Island (OR ≈ 0.14) are much lower.
- **Irregular lots max out more often.** The Department of Finance classifies corner lots as irregular, and corner lots are often the most desirable sites.
- **Land value has a positive but modest effect.** Price alone doesn't explain how intensively a lot is built.
- **Two predictors are not significant:** landmark status and Manhattan (relative to the Bronx). Their bootstrap confidence intervals include 0.
- **Trends over time:** utilization rose steadily after ZQA and peaked around 2021. It fell to about 0.90 after COVID, then had a lagged rise and fall around the 421-a expiration, reaching a local peak in mid-2024.

### Policy recommendations

1. Make it easier to demolish and replace existing buildings, since alterations rarely use their full zoning capacity.
2. Encourage more density in Queens and Staten Island through density bonuses and lower parking requirements.
3. Keep tax incentives for affordable housing (such as 485-x) running without gaps, so there is no repeat of the 421-a lapse.

See `Project_Report.pdf` for the full discussion.

---

## Reproducing the analysis

### Requirements

R (≥ 4.0) and the following packages:

```r
install.packages(c(
  "ggplot2", "dplyr", "readr", "lubridate", "reshape2",
  "car", "gridExtra", "stringr", "boot", "mgcv", "scales"
))
```

### Steps

1. Clone the repository and place the four CSVs listed above in one folder.
2. In `Project_Code.R`, set the working directory on line 9 to that folder:
   ```r
   setwd("path/to/your/data")
   ```
3. Run the script from top to bottom.

### Notes and caveats

- **Add `library(boot)`** before the bootstrap section. The script calls `boot()` but doesn't load the package.
- **Memory:** the bootstrap and the GAM plot both need a lot of RAM. They were run in a high-RAM Google Colab environment.
- **Bootstrap resamples:** the script sets `R = 1000`, but the report's results used **100 resamples** because of the dataset's size. Set `R = 100` to reproduce the reported intervals, or keep 1000 for tighter estimates if you have the memory.
- Ratios above 3 are dropped from the time-series plot, which is restricted to 2012-01-01 through 2025-11-01.

---

## Sources

- NYC Open Data: DOB Certificate of Occupancy, DOB NOW: Certificate of Occupancy, DOB Job Application Filings and PLUTO
- NYC Rent Guidelines Board, *Housing Supply Report*
- Douglas Elliman, Manhattan rental market reports
- NYC Department of City Planning, *Zoning for Quality and Affordability*
