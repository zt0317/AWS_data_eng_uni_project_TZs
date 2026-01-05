## Data preparation:

## 1. FluView API: 

### Official CDC FluView

### https://www.cdc.gov/fluview/index.html 

FluView is an official surveillance platform maintained by the U.S. Centers for Disease Control and Prevention (CDC).
It provides weekly aggregated epidemiological data on influenza cases in the US, providing information about: influenza rates, confirmed cases, virus subtype distributions, hospitalization and mortality rates.
However, there is no available public REST API due to security reasons.

### Delphi Epidata API (FluView Access)

### https://cmu-delphi.github.io/delphi-epidata/api/fluview.html

To address this limitation, the Delphi Group provides a proxy solution via the Delphi Epidata API, which retrieves and standardizes FluView data directly from CDC sources.

### The requested information:

| Field | Description | Type |
|------|-------------|------|
| epidata[].location | Name of the catchment (e.g. `network_all`, `CA`, `NY_albany`) | string |
| epidata[].epiweek | Epiweek during which the data was collected | integer |
| epidata[].rate_age_0 | Hospitalization rate for ages 0–4 | float |
| epidata[].rate_age_1 | Hospitalization rate for ages 5–17 | float |
| epidata[].rate_age_2 | Hospitalization rate for ages 18–49 | float |
| epidata[].rate_age_3 | Hospitalization rate for ages 50–64 | float |
| epidata[].rate_age_4 | Hospitalization rate for ages 65+ | float |
| epidata[].rate_overall | Overall hospitalization rate | float |
| epidata[].rate_age_5 | Hospitalization rate for ages 65–74 | float |
| epidata[].rate_age_6 | Hospitalization rate for ages 75–84 | float |
| epidata[].rate_age_7 | Hospitalization rate for ages 85+ | float |
| epidata[].rate_age_18t29 | Hospitalization rate for ages 18–29 | float |
| epidata[].rate_age_30t39 | Hospitalization rate for ages 30–39 | float |
| epidata[].rate_age_40t49 | Hospitalization rate for ages 40–49 | float |
| epidata[].rate_age_5t11 | Hospitalization rate for ages 5–11 | float |
| epidata[].rate_age_12t17 | Hospitalization rate for ages 12–17 | float |
| epidata[].rate_age_lt18 | Hospitalization rate for ages <18 | float |
| epidata[].rate_age_gte18 | Hospitalization rate for ages ≥18 | float |
| epidata[].rate_age_0tlt1 | Hospitalization rate for ages 0–1 | float |
| epidata[].rate_age_1t4 | Hospitalization rate for ages 1–4 | float |
| epidata[].rate_age_gte75 | Hospitalization rate for ages ≥75 | float |


## 2. Air Quality Data

### https://www.kaggle.com/datasets/samithsachidanandan/air-quality-data-2019-2025

Kaggle dataset:

*"This is a synthetic dataset generated using python library faker. This dataset includes the air quality data of all states of USA from 2019 - 2025."*

Since the dataset is synthetic it may not serve purposeful insights, but this dataset was well aggragetad in the preferred time interval (2020-2025).

## Columns:

- date	
- city	
- latitude	
- longitude	
- pm25	
- pm10	
- o3	
- no2	
- so2	
- co	
- aqi	
- temperature_c	
- humidity_percent	
- wind_speed_mps	
- month	
- day	
- year	
- day_of_week	
- hour	
- is_weekend	
- day_of_year		
- AQI_Category

All these columns not needed, and it is hourly data

**! Weekly aggregation is needed for simplicity !**

### Weekly aggregation

    weekly_df_aux = df.groupby(['city', 'year_week'])[['season', 'AQI_Category']].agg(lambda x: x.mode()[0]).reset_index()
    weekly_df = weekly_df.merge(weekly_df_aux, on=['city', 'year_week'])

### Dropping the unnecessary columns

    weekly_df = weekly_df.drop(columns=["latitude", "longitude", "pm10", "o3", 
                                    "no2", "so2", "co", "humidity_percent", 
                                    "wind_speed_mps", "season"])

### Final Dataset ready for the S3 bucket:

| city   | state    | state_abbr | year_week | pm25       | aqi       | temperature_c | AQI_Category |
|--------|----------|------------|-----------|------------|-----------|---------------|--------------|
| Albany | New York | NY         | 202000    | 11.017500  | 44.500000 | 17.322500     | Good         |
| Albany | New York | NY         | 202001    | 10.972857  | 51.571429 | 17.292857     | Moderate     |
| Albany | New York | NY         | 202002    | 12.811429  | 42.571429 | 12.054286     | Good         |
| Albany | New York | NY         | 202003    | 11.981429  | 48.857143 | 16.432857     | Good         |
| Albany | New York | NY         | 202004    | 13.565714  | 50.000000 | 15.512857     | Moderate     |