# README: Visualizing U.S. Labor Force Participation with IPUMS CPS

## Project Overview

This project explores how labor-force participation in the United States varies across age, gender, and educational attainment. It was completed for AEDS6400 Blog Post 3, which emphasizes using data visualization to tell a coherent story about the U.S. labor market.

Rather than beginning with a complicated statistical model, I chose a descriptive question that could be investigated with CPS microdata and communicated through several complementary visualizations:

**Research question:** How does labor-force participation vary across age, gender, and educational attainment in the United States?

The main deliverable is an English-language blog post with three primary figures. Additional figures are included in an appendix as supplementary descriptive visualizations.

## Project Workflow

### 1. Choosing a research question

The assignment encouraged students to explore an interesting U.S. labor-market question and present at least three substantive visualizations. Since the project emphasized visualization and interpretation more than regression, I selected labor-force participation as the outcome and age, gender, and education as the main dimensions of comparison.

This question was suitable for a descriptive project because:
- labor-force participation has a clear economic interpretation;
- the selected demographic variables are available in CPS microdata;
- age, gender, and education provide several related comparisons;
- the same outcome can be presented with different chart types.

The figures were designed to build on one another: first describe the age pattern, then compare age patterns by gender, and finally examine age and education together.

### 2. Finding and downloading the data

I used **IPUMS CPS**, which provides harmonized microdata from the U.S. Current Population Survey.

Website: https://cps.ipums.org/cps/

The workflow was:

1. Open IPUMS CPS and create or access an account.
2. Start a new data extract.
3. Select the CPS samples needed for the analysis.
4. Select the variables required to calculate labor-force participation and group observations.
5. Choose a person-level, rectangular extract in comma-delimited CSV format.
6. Submit the extract, wait for it to be prepared, and download the resulting compressed CSV file.

The downloaded file was a `.csv.gz` file. It did not need to be manually unzipped: R can read a gzip-compressed CSV directly.

### 3. Selecting variables

The extract used these seven variables:

| Variable | Purpose |
|---|---|
| `YEAR` | Identifies the survey year |
| `MONTH` | Identifies the survey month |
| `AGE` | Allows the sample to be restricted to age 16 and older and grouped by age |
| `SEX` | Distinguishes male and female respondents |
| `EDUC` | Measures educational attainment |
| `LABFORCE` | Identifies labor-force status |
| `WTFINL` | Person-level final weight used for Basic Monthly CPS estimates |

I kept the extract focused on variables needed for this descriptive analysis. IPUMS variable documentation was used to check the meaning and coding of the variables, especially `LABFORCE`, `EDUC`, and `WTFINL`.

Useful documentation:
- IPUMS CPS: https://cps.ipums.org/cps/
- `WTFINL`: https://cps.ipums.org/cps-action/variables/WTFINL
- `EDUC`: https://cps.ipums.org/cps-action/variables/educ
- `LABFORCE`: https://cps.ipums.org/cps-action/variables/169912

### 4. Importing the data into R

I used R and the `tidyverse` package to import, clean, summarize, and visualize the data.

```r
library(tidyverse)

cps <- read_csv(
  "path/to/cps_00002.csv.gz"
)
```

The file contained observations from 2004 through 2026, even though the main analysis was intended to focus on recent years. I inspected the years and months before deciding on the analysis window.

### 5. Defining the analysis period

I restricted the main analysis to **2020–2025**, six complete calendar years in the downloaded extract. The 2026 observations only covered January through August in this extract, so I excluded 2026 to avoid mixing complete years with an incomplete year.

```r
cps <- cps %>%
  filter(YEAR >= 2020, YEAR <= 2025)
```

The estimates in the blog therefore describe pooled CPS observations from 2020 through 2025. They should not be interpreted as estimates for 2025 alone or as a time trend.

### 6. Cleaning the analytic sample

For the labor-force participation calculations, I restricted the data to people age 16 and older, observations with a valid labor-force status, and positive, non-missing person weights.

In the IPUMS `LABFORCE` coding used here:
- `1` means not in the labor force;
- `2` means in the labor force;
- `0` represents an inapplicable/not-in-universe category for this analysis.

I excluded `LABFORCE == 0` from the denominator rather than treating it as “not in the labor force.” I also excluded missing or non-positive `WTFINL` values from the weighted estimates.

```r
cps_clean <- cps %>%
  filter(
    AGE >= 16,
    LABFORCE %in% c(1, 2),
    !is.na(WTFINL),
    WTFINL > 0
  ) %>%
  mutate(
    in_labor_force = if_else(LABFORCE == 2, 1, 0)
  )
```

The `in_labor_force` variable is a binary indicator: 1 for people in the labor force and 0 for people not in the labor force, among observations with a valid labor-force status.

### 7. Creating age and education groups

For the first two figures and the heatmap, I grouped age into six categories:

- 16–24
- 25–34
- 35–44
- 45–54
- 55–64
- 65+

```r
cps_clean <- cps_clean %>%
  mutate(
    age_group = case_when(
      AGE < 25 ~ "16–24",
      AGE < 35 ~ "25–34",
      AGE < 45 ~ "35–44",
      AGE < 55 ~ "45–54",
      AGE < 65 ~ "55–64",
      TRUE ~ "65+"
    ),
    age_group = factor(
      age_group,
      levels = c("16–24", "25–34", "35–44",
                 "45–54", "55–64", "65+")
    )
  )
```

I also created a male/female label from `SEX` and grouped the available `EDUC` codes into four broad educational-attainment categories:
- Less than high school
- High school diploma
- Some college / associate
- Bachelor's degree or higher

The broad groups make the results easier to compare visually than the original detailed education codes. The exact code mapping should be checked against the IPUMS `EDUC` codebook when reproducing or extending the analysis.

### 8. Calculating weighted labor-force participation rates

The main outcome is the labor-force participation rate (LFPR). Because CPS is a sample survey, I used the person-level final weight `WTFINL` rather than calculating an unweighted sample proportion.

For a group of eligible respondents, the weighted LFPR is:

\[
\mathrm{LFPR}
=
\frac{\sum_i w_i \,\mathrm{InLaborForce}_i}
{\sum_i w_i}
\times 100,
\]

where \(w_i\) is `WTFINL`, and `InLaborForce` equals 1 for respondents in the labor force and 0 for respondents not in the labor force.

In R, the calculation was implemented with `group_by()` and `summarise()`. The same weighted formula was applied separately by age group, by age group and gender, and by age group and education group.

These are descriptive weighted estimates based on the selected IPUMS CPS sample and the stated cleaning rules. They are not causal estimates, and they should not automatically be treated as exact reproductions of published Bureau of Labor Statistics estimates.

### 9. Building the visualizations

I used `ggplot2` to create figures with different visual forms, selected to match the comparison being made.

#### Chart 1: Labor-force participation by age group

- **Chart type:** Bar chart (`geom_col()`)
- **Data:** `lfpr_age`
- **Purpose:** Establishes the overall relationship between age group and weighted LFPR.

#### Chart 2: Labor-force participation by age and gender

- **Chart type:** Line chart with points (`geom_line()` and `geom_point()`)
- **Data:** `lfpr_age_sex`
- **Purpose:** Compares male and female participation patterns across age groups.

#### Chart 3: Labor-force participation by age and education

- **Chart type:** Heatmap (`geom_tile()`), with percentage labels (`geom_text()`)
- **Data:** `lfpr_age_edu`
- **Purpose:** Displays weighted LFPR for each age-by-education combination. Color encodes the rate, while labels make the values readable.

The three main charts form a sequence: overall age differences, gender differences across age, and education differences across age.

### 10. Additional visualizations in the appendix

I also explored additional chart types as supplementary figures:

- **Chart 4:** Boxplot of age by educational-attainment group (`geom_boxplot()`).
- **Chart 5:** Dot plot of LFPR by education and age (`geom_point()`), using the already summarized `lfpr_age_edu` data.
- **Chart 6:** Weighted age-density plot by gender (`geom_density()`), using `WTFINL` as the weight.

These figures demonstrate additional `ggplot2` techniques and provide alternative views of the data. They are supplementary rather than necessary to the main argument.

### 11. Writing the blog and assembling the document

After creating and checking the figures, I organized the blog post around one central question. The main text introduces the data and method, discusses Charts 1–3 in order, and concludes by summarizing the descriptive patterns without claiming causation.

The figures were exported from R as image files and inserted into a LaTeX document using `\includegraphics`. The three main figures appear in the body of the blog, while Charts 4–6 are placed in an appendix with brief captions.

## Reproducibility Notes

To reproduce the project:

1. Obtain an IPUMS CPS extract containing the seven variables listed above.
2. Update the file path in `read_csv()` to match the downloaded file.
3. Run the filtering, cleaning, and recoding steps in order.
4. Calculate the weighted summaries.
5. Run the `ggplot2` code for each figure.
6. Export the figures with the filenames expected by the LaTeX document (for example, `chart1.png` through `chart6.png`).
7. Compile the LaTeX document.

The results depend on the exact extract, sample selection, variable coding, and cleaning rules. If the IPUMS extract changes, the summaries and figures should be regenerated rather than assuming the values will remain identical.

## Limitations

This is a descriptive visualization project, not a causal study. Age, gender, and educational attainment may be associated with labor-force participation, but the figures alone cannot establish why the differences occur. The analysis pools observations from 2020–2025, so it does not describe year-by-year changes. In addition, education categories and the restriction to valid labor-force status and positive weights affect the analytic sample. These choices are documented so readers can understand how the results were produced.

## Files

A suggested project structure is:

```text
project/
├── README.md
├── main.tex
├── chart1.png
├── chart2.png
├── chart3.png
├── chart4.png
├── chart5.png
├── chart6.png
└── cps_00002.csv.gz
```

The CPS extract may be large and subject to IPUMS terms or access restrictions. Consider keeping the raw data out of a public repository and instead documenting how to obtain it from IPUMS.

## Acknowledgments and Data Source

Data: IPUMS Current Population Survey (CPS), University of Minnesota.

IPUMS CPS website: https://cps.ipums.org/cps/

This README documents the workflow and analytical choices used to complete the visualization project. It emphasizes the steps from research-question selection and data acquisition through cleaning, weighted summaries, visualization, and final presentation.
