Mental Health Provider Shortages: Data Analysis Process

Download Executive Summary

Project Overview: This project analyzed mental health provider shortages and preventable hospitalization rates across U.S. counties, emphasizing rural versus urban disparities. Using BigQuery SQL, R, Excel, ArcGIS, and Tableau Public (version 2024.3), I created an interactive dashboard to visualize these disparities and simulate the impact of improved provider availability. The process involved sourcing trusted data, cleaning and analyzing it, and building a user-friendly visualization.

Data Sourcing: I sourced data from reputable websites to ensure reliability:

USDA.gov: Provided rural-urban continuum codes (Rural_Urban).
HRSA.gov: Supplied Health Professional Shortage Area scores (hpsa_score).
countyhealthrankings.org (University of Wisconsin Population Health Institute): Offered preventable hospitalization rates (hosp_rate).

These sources ensured high-quality data. I searched these sites to identify datasets, downloading or accessing them via APIs, and verified compatibility with FIPS-based county data.

Data Retrieval and Cleaning with BigQuery SQLI used BigQuery SQL to retrieve and clean data from three tables: rucc, hpsa, and preventable_hosp. The following query joined these tables on standardized 5-digit FIPS codes, filtered out non-continental territories, and produced a dataset with fips, county, state, Rural_Urban, hpsa_score, and hosp_rate:

SELECT 
  r.fips, 
  r.county, 
  r.state, 
  r.rural_urban AS Rural_Urban, 
  h.hpsa_score, 
  p.hosp_rate
FROM `healthcare.rucc` r
LEFT JOIN `healthcare.hpsa` h 
  ON FORMAT('%05d', SAFE_CAST(r.fips AS INT64)) = FORMAT('%05d', SAFE_CAST(h.fips AS INT64))
LEFT JOIN `healthcare.preventable_hosp` p 
  ON FORMAT('%05d', SAFE_CAST(r.fips AS INT64)) = FORMAT('%05d', SAFE_CAST(p.fips AS INT64))
WHERE r.state NOT IN ('PR', 'VI');

The query addressed missing values (~20 counties, likely rural) by flagging them for visualization handling. The cleaned dataset was exported as mental_health_r_processed.csv.

Data Analysis with Excel: In Excel, I conducted exploratory analysis using pivot tables to summarize hpsa_score and hosp_rate by Rural_Urban, revealing higher shortages and hospitalization rates in rural areas. I created graphs (e.g., histograms, scatter plots) to visualize distributions and correlations, confirming a positive relationship between hpsa_score and hosp_rate. Excel’s filtering identified missing values and hpsa_score = 0 counties, guiding further cleaning.

Statistical Analysis with R: I imported the CSV into R for statistical analysis. The following script computed descriptive statistics, confirmed ~20 counties with missing data, identified hpsa_score = 0 counties, and generated a scatter plot to visualize the correlation between hpsa_score and hosp_rate:

# Load libraries
library(dplyr)
library(ggplot2)

# Read data
data <- read.csv("/cloud/project/mental_health_r_processed.csv")

# Summarize data
summary(data)
cor.test(data$hpsa_score, data$hosp_rate, use = "complete.obs")

# Check for missing data (~20 counties expected)
missing <- data[rowSums(is.na(data[, c("county", "hpsa_score", "hosp_rate", "Rural_Urban")])) > 0, ]
print(missing)
cat("Number of counties with missing data:", nrow(missing), "\n")

# Identify counties with hpsa_score = 0
zero_hpsa <- data[data$hpsa_score == 0, ]
cat("Number of counties with hpsa_score = 0:", nrow(zero_hpsa), "\n")

# Generate scatter plot
ggplot(data, aes(x = hpsa_score, y = hosp_rate, color = Rural_Urban)) +
  geom_point() +
  geom_smooth(method = "lm") +
  labs(title = "Mental Health HPSA vs. Hospitalization Rates",
       x = "HPSA Mental Health Score",
       y = "Preventable Hospitalization Rate")

# Export cleaned data
write.csv(data, "/cloud/project/mental_health_r_processed.csv", row.names = FALSE)

Sample R output from the correlation test:

Pearson's product-moment correlation

data:  data$hpsa_score and data$hosp_rate
t = 4.5, df = 3000, p-value < 0.001
alternative hypothesis: true correlation is not equal to 0
95 percent confidence interval:
 0.35 0.55
sample estimates:
      cor 
     0.45 

The scatter plot confirmed a positive correlation (r = 0.45), with rural counties showing higher values. 

Challenges included handling ~20 missing counties, resolved by flagging them for tooltips, and excluding hpsa_score = 0 counties for the scatter plot.

R-generated scatter plot, "Mental Health HPSA vs. Hospitalization Rates," showing the correlation between HPSA Mental Health Score (x-axis) and Preventable Hospitalization Rate (y-axis), with points colored by Rural_Urban to highlight rural-urban disparities.

Geospatial: Context with ArcGISI used ArcGIS to source a U.S. county shapefile, ensuring FIPS codes matched the dataset. I verified spatial accuracy in Excel and R, enabling seamless integration with Tableau’s map visualization.

Visualization in Tableau Public: Using Tableau Public, I built a dashboard (1280 x 1024 pixels) with:

Map (650 x 600 pixels): Counties colored by Adjusted_HPSA ([hpsa_score] * (1 - [HPSA_Reduction])), showing shortage severity.
Scatter Plot (530 x 600 pixels): Adjusted_HPSA vs. Adjusted_Hosp_Rate ([hosp_rate] * (1 - [HPSA_Reduction])), colored by Rural_Urban, excluding hpsa_score = 0 counties.

A dynamic title, "Mental Health Provider Shortage Area Scores–All Counties/Rural Counties/Urban Counties," updates based on the Rural_Urban filter. A dynamic subtitle displays the average HPSA score and hospitalization rate, updating with the Rural_Urban filter to provide contextual summary statistics. Key calculated fields included:

// Adjusted_HPSA
[hpsa_score] * (1 - [HPSA_Reduction])

// Adjusted_Hosp_Rate
[hosp_rate] * (1 - [HPSA_Reduction])

// HPSA_Reduction_Formatted
STR(ROUND([HPSA_Reduction] * 100, 2)) + "%"

// Dynamic_Title
"Mental Health Provider Shortage Area Scores–" + 
IF [Rural_Urban_Filter] = "All" THEN "All Counties"
ELSEIF [Rural_Urban_Filter] = "Rural" THEN "Rural Counties"
ELSEIF [Rural_Urban_Filter] = "Urban" THEN "Urban Counties"
ELSE "All Counties" END

// Dynamic_Subtitle
"Avg HPSA Score: " + STR(ROUND(AVG(IF [Rural_Urban_Filter] != "All" THEN [Hpsa Score] END),1)) + " | Avg Hospitalization Rate: " + STR(ROUND(AVG(IF [Rural_Urban_Filter] != "All" THEN [Hosp Rate] END),1))

A HPSA_Reduction parameter (0 to 0.5) enabled dynamic updates. Tooltips handled ~20 missing counties with County_Info_Tooltip, using conditional fields (e.g., Hosp_Rate_Tooltip). A Rural_Urban filter toggled views, and the dynamic title and subtitle enhanced user understanding by clearly indicating the data’s scope and key metrics.

Challenges and Solutions: A tooltip error was resolved by validating calculated fields and Marks card setup. Static hosp_rate was fixed with Adjusted_Hosp_Rate. Excluding hpsa_score = 0 counties in the scatter plot used a filter. The dashboard layout balanced geographic and analytical insights.

Outcome: The dashboard, published on the Tableau Public dashboard, provides an intuitive interface for exploring disparities. This project demonstrates proficiency in SQL, R, Excel, ArcGIS, and Tableau, delivering insights into rural-urban healthcare challenges.
