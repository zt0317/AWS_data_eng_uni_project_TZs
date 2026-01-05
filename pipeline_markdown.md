## AWS PIPELINE

### From the original idea, I modified the data pipeline, I added cloud9 aswell

<p align="center">
<img src="Képernyőfotó 2026-01-04 - 19.15.37.png" alt="S3 Folders" width="500">
</p>


### 1. S3 buckets:

**Create S3 bucket for the used data sources:**

<p align="center">
<img src="Képernyőfotó 2026-01-04 - 18.03.30.png" alt="S3 Folders" width="500">
</p>

- Uploaded the air quality csv

<p align="center">
<img src="Képernyőfotó 2026-01-04 - 18.08.10.png" alt="S3 Folders" width="500">
</p>

- Created a query results bucket for later Athena queries

<p align="center">
<img src="Képernyőfotó 2026-01-04 - 18.10.07.png" alt="S3 Folders" width="300">
</p>

### 2. Lambda function:

Put it into a csv format, query it from 2020-2025 - these years crucial in case of respiratory diseases

    import json
    import boto3
    import urllib.parse
    import urllib.request
    import csv
    from io import StringIO

    s3 = boto3.client("s3")

    BUCKET = "air-flu-bucket"
    KEY = "flu_surveillance/flu_surveillance_2020_2025.csv"

    API_KEY = "289703e85720a"

    states = [
        'NY','MD','GA','ME','TX','LA','ND','ID','MA','NV','WV','WY','SC','OH',
        'NH','CO','IA','DE','KY','PA','CT','MT','HI','IN','MS','MO','AK','MI',
        'NE','AR','WI','AL','VT','TN','OK','WA','AZ','SD','RI','NC','VA','CA',
        'MN','OR','UT','NM','IL','FL','KS','NJ','DC'
    ]

    epiweeks_range = "202001-202530"

    API_URL = "http://api.delphi.cmu.edu/epidata/api.php"

    def lambda_handler(event, context):

        params = {
            "source": "flusurv",
            "locations": ",".join(states),
            "epiweeks": epiweeks_range,
            "api_key": API_KEY
        }

        request_url = f"{API_URL}?{urllib.parse.urlencode(params)}"

        req = urllib.request.Request(
            request_url,
            headers={"User-Agent": "Mozilla/5.0"}
        )

        with urllib.request.urlopen(req) as response:
            data = json.loads(response.read())

        if data.get("result") != 1:
            raise Exception(f"API error: {data.get('message')}")

        records = data.get("epidata", [])

        age_cols = [k for k in records[0].keys() if k.startswith("rate_age")]
        fieldnames = ["location", "epiweek"] + age_cols

        csv_buffer = StringIO()
        writer = csv.DictWriter(csv_buffer, fieldnames=fieldnames)
        writer.writeheader()

        for row in records:
            writer.writerow({k: row.get(k) for k in fieldnames})

        s3.put_object(
            Bucket=BUCKET,
            Key=KEY,
            Body=csv_buffer.getvalue()
        )

        return {
            "statusCode": 200,
            "body": f"Stored {len(records)} records in S3"
        }

<p align="center">
<img src="Képernyőfotó 2025-12-29 - 23.12.23.png" alt="S3 Folders" width="500">
</p>

<p align="center">
<img src="Képernyőfotó 2025-12-29 - 23.13.19.png" alt="S3 Folders" width="500">
</p>

- Created a test event to see whether it was successful

<p align="center">
<img src="Képernyőfotó 2025-12-29 - 23.16.07.png" alt="S3 Folders" width="500">
</p>

<p align="center">
<img src="Képernyőfotó 2025-12-29 - 23.17.51.png" alt="S3 Folders" width="500">
</p>

- It worked - here is the csv in the right bucket-folder

<p align="center">
<img src="Képernyőfotó 2026-01-04 - 18.18.03.png" alt="S3 Folders" width="500">
</p>

### 3. Cloud9 IDE for csv handling

**Created a Cloud9 environment for the handling of the csv columns (inspired by the capstone project)**

<p align="center">
<img src="Képernyőfotó 2026-01-04 - 12.44.11.png" alt="S3 Folders" width="700">
</p>

<p align="center">
<img src="Képernyőfotó 2026-01-04 - 12.44.45.png" alt="S3 Folders" width="700">
</p>

    import pandas as pd

    df = pd.read_csv('flu_surveillance_2020_2025.csv')

    df.rename(columns={
        'location': 'state_abbr',
        'epiweek': 'year_week'
    }, inplace=True)

    df.to_csv('flu_surveillance_2020_2025.csv', index=False)

    print("Columns renamed successfully!")

**Then I updated the original csv**

<p align="center">
<img src="Képernyőfotó 2026-01-04 - 12.55.03.png" alt="S3 Folders" width="700">
</p>

### 4. Create database for the tables:

**I created a database where I can store the tables from the datasets**

<p align="center">
<img src="Képernyőfotó 2025-12-29 - 23.19.25.png" alt="S3 Folders" width="500">
</p>

### 5. Athena - Table creation

*Instead of crawlers, this time I created tables with SQL commands*

**Air Quality Data:**

    CREATE EXTERNAL TABLE air_quality_csv (
    city STRING,
    state STRING,
    state_abbr STRING,
    year_week BIGINT,
    pm25 DOUBLE,
    aqi DOUBLE,
    temperature_c DOUBLE,
    AQI_Category STRING
    )

    ROW FORMAT SERDE 'org.apache.hadoop.hive.serde2.OpenCSVSerde'
    WITH SERDEPROPERTIES (
    'separatorChar' = ',',
    'quoteChar' = '"'
    )
    LOCATION 's3://air-flu-bucket/air_quality/' 
    TBLPROPERTIES (
    'skip.header.line.count'='1',
    'use.null.for.invalid.data'='true'
    );

It points to the folder where the Air quality csv can be found

<p align="center">
<img src="Képernyőfotó 2026-01-04 - 18.45.30.png" alt="S3 Folders" width="700">
</p>

**Flu Surveillance**

    CREATE EXTERNAL TABLE flu_surveillance_2020_2025_csv (
    state_abbr STRING,
    year_week BIGINT,
    rate_age_0 DOUBLE,
    rate_age_1 DOUBLE,
    rate_age_2 DOUBLE,
    rate_age_3 DOUBLE,
    rate_age_4 DOUBLE,
    rate_age_5 DOUBLE,
    rate_age_6 DOUBLE,
    rate_age_7 DOUBLE,
    rate_age_18t29 DOUBLE,
    rate_age_30t39 DOUBLE,
    rate_age_40t49 DOUBLE,
    rate_age_5t11 DOUBLE,
    rate_age_12t17 DOUBLE,
    rate_age_lt18 DOUBLE,
    rate_age_gte18 DOUBLE,
    rate_age_1t4 DOUBLE,
    rate_age_gte75 DOUBLE
    )
    ROW FORMAT SERDE 'org.apache.hadoop.hive.serde2.OpenCSVSerde'
    WITH SERDEPROPERTIES (
    'separatorChar' = ',',
    'quoteChar' = '"'
    )
    LOCATION 's3://air-flu-bucket/flu_surveillance/'
    TBLPROPERTIES (
    'skip.header.line.count'='1',
    'use.null.for.invalid.data'='true'
    );

<p align="center">
<img src="Képernyőfotó 2026-01-04 - 18.47.31.png" alt="S3 Folders" width="700">
</p>

They appeared as tables in the database:

<p align="center">
<img src="Képernyőfotó 2026-01-04 - 11.45.42.png" alt="S3 Folders" width="700">
</p>


### 6. Athena - Querying data

**Joined View Creation**

    SELECT
        state_abbr,
        year_week,
        aqi,
        AQI_Category,
        LAG(aqi, 1) OVER (PARTITION BY state_abbr ORDER BY year_week) AS aqi_lag_1w,
        rate_age_gte18
    FROM air_flu_joined;

Created a joined view between the two tables on the same columns. This made later queries so much easier, there is no need to join the tables together each time

<p align="left">
<img src="Képernyőfotó 2026-01-04 - 13.49.51.png" alt="S3 Folders" width="500">
</p>

**Yearly aggregation**

    SELECT
        CAST(SUBSTR(CAST(year_week AS VARCHAR), 1, 4) AS INTEGER) AS year,
        AVG(pm25) AS avg_pm25,
        AVG(aqi) AS avg_aqi,
        AVG(rate_age_lt18) AS avg_rate_lt18,
        AVG(rate_age_gte18) AS avg_rate_gte18,
        AVG(rate_age_gte75) AS avg_rate_gte75
    FROM air_flu_joined
    GROUP BY CAST(SUBSTR(CAST(year_week AS VARCHAR), 1, 4) AS INTEGER)
    ORDER BY year;

<p align="left">
<img src="Képernyőfotó 2026-01-04 - 18.55.33.png" alt="S3 Folders" width="700">
</p>

<span style="color:red; font-weight:bold;">
This query extracts the year from the year_week variable and calculates the yearly average values of pm_2.5, aqi, and flu rates for different age groups. This aggregation allows for a comparison of environmental and health indicators across years. In 2025 there was a noticeable increase in both pm_2.5 and flu_rates, while they remained relatively stable between 2020 and 2024. The increase in flu rates is especially among people aged over 75, with an average flu rate of 30.61. This suggests that older age groups may have been particularly affected during this period.
</span>

#

**Correlation**

    SELECT
        state_abbr,
        corr(aqi, rate_age_0) AS corr_aqi_age_0,
        corr(aqi, rate_age_1) AS corr_aqi_age_1,
        corr(aqi, rate_age_2) AS corr_aqi_age_2,
        corr(aqi, rate_age_3) AS corr_aqi_age_3,
        corr(aqi, rate_age_4) AS corr_aqi_age_4,
        corr(aqi, rate_age_5) AS corr_aqi_age_5,
        corr(aqi, rate_age_6) AS corr_aqi_age_6,
        corr(aqi, rate_age_7) AS corr_aqi_age_7
    FROM air_flu_joined
    GROUP BY state_abbr
    ORDER BY corr_aqi_age_7 DESC;
#

    SELECT
        state_abbr,
        corr(aqi,  rate_age_lt18) AS corr_aqi_age_lt18,
        corr(aqi, rate_age_gte18) AS corr_aqi_age_gte18,
        corr(aqi, rate_age_gte75) AS corr_aqi_age_gte75
        
    FROM air_flu_joined
    GROUP BY state_abbr
    ORDER BY corr_aqi_age_gte75 DESC;

<p align="left">
<img src="Képernyőfotó 2026-01-04 - 14.24.55.png" alt="S3 Folders" width="500">
</p>

<span style="color:red; font-weight:bold;">This query calculates the correlation between the Air Quality Index and flu rates for three age groups, grouped by state, to see how air quality relates to influenza outbreaks. Overall, the correlations are weak, with most states showing values near zero, indicating little general relationship between AQI and flu rates (but the data is synthetic). Some state-specific differences appear: North Carolina shows the highest positive correlation for the elderly (0.267), while Iowa shows a slight negative correlation (-0.106). Positive correlations tend to be stronger in older adults than in children, as seen in Michigan and Oregon. Even though overall correlations are low, this analysis is useful for spotting anomalies. Sudden spikes in correlation can highlight states or periods where air pollution may significantly affect respiratory health or where other factors, like extreme weather play a role.
</span>

**Weeks with high AQI values**

Highest measure was about ~100 and the lowest ~25, so I figured the outliers are above 85

    SELECT
        state_abbr,
        year_week,
        aqi,
        rate_age_lt18,
        rate_age_gte18,
        rate_age_gte75
        
    FROM air_flu_joined
    WHERE aqi >= 85
    ORDER BY aqi DESC;

<p align="left">
<img src="Képernyőfotó 2026-01-04 - 14.22.13.png" alt="S3 Folders" width="500">
</p>

<span style="color:red; font-weight:bold;">This query filters the dataset for "outlier" weeks where the AQI was 85 or higher to see if extreme pollution events triggered immediate flu spikes. Interestingly, many of the highest AQI weeks (in MN or CT) show very low flu rates (0.0 to 0.6), suggesting that isolated spikes in air pollution do not immediately result in a surge of flu cases. (Synthetic data!)</span>

**Seasonality**

    SELECT
        state_abbr,
        CASE
            WHEN (year_week % 100) BETWEEN 48 AND 52
            OR (year_week % 100) BETWEEN 1 AND 4
            THEN 'Winter'
            ELSE 'Non-Winter'
        END AS season,
        AVG(aqi) AS avg_aqi,
        AVG(rate_age_lt18) AS avg_flu_children,
        AVG(rate_age_gte75) AS avg_flu_elderly
    FROM air_flu_joined
    GROUP BY
        state_abbr,
        CASE
            WHEN (year_week % 100) BETWEEN 48 AND 52
            OR (year_week % 100) BETWEEN 1 AND 4
            THEN 'Winter'
            ELSE 'Non-Winter'
        END
    ORDER BY state_abbr, season;

<p align="left">
<img src="Képernyőfotó 2026-01-04 - 20.28.38.png" alt="S3 Folders" width="500">
</p>

<span style="color:red; font-weight:bold;">The code categorizes the data into two groups “Winter,” which includes weeks 48 to 52 and 1 to 8, and “Non-Winter,” which includes all other weeks. This allows for a comparison of environmental factors with health outcomes across different seasons. The results show that the Air Quality Index (AQI) remains fairly consistent between Winter and Non-Winter periods. However, flu rates for both children and the elderly are much higher during the Winter months. This indicates that the timing of the season has a much stronger effect on flu rates than air quality does.</span>

**Children and Elderly with AQI levels**

    SELECT
        state_abbr,
        AQI_Category AS aqi_level,
        AVG(rate_age_lt18) AS avg_flu_children,
        AVG(rate_age_gte75) AS avg_flu_elderly
    FROM air_flu_joined
    GROUP BY state_abbr, AQI_Category
    ORDER BY state_abbr, AQI_Category;

<p align="left">
<img src="Képernyőfotó 2026-01-04 - 14.28.18.png" alt="S3 Folders" width="500">
</p>

<span style="color:red; font-weight:bold;">The SQL query analyzes the relationship between air quality categories (AQI levels) and average flu rates for children and the elderly across different states, two vulnerable categories. The results show that the elderly population consistently has much higher flu rates than children, regardless of air quality. The data does not show a consistent increase in flu rates as air quality worsens. These results also help identify unusual patterns. For instance, Iowa recorded a 0.0 flu rate for both children and the elderly during “Moderate” air quality weeks, which is a clear outlier compared to other states, that could be due to technical reasons.</span>

### 7. Visualisation with Cloud9:

    import boto3
    import pandas as pd
    import matplotlib.pyplot as plt
    from io import StringIO

    BUCKET = 'air-flu-athena-results'
    KEY = 'simple_query/2026/01/04/6a0bc910-3534-477b-84ea-6c55c02c4300.csv'

    s3 = boto3.client('s3')
    obj = s3.get_object(Bucket=BUCKET, Key=KEY)
    csv_data = obj['Body'].read().decode('utf-8')

    df = pd.read_csv(StringIO(csv_data))

    plt.figure(figsize=(10,6))
    plt.scatter(df['pm25'], df['rate_age_lt18'], alpha=0.7, c='green')
    plt.xlabel('PM2.5')
    plt.ylabel('Flu Rate (<18)')
    plt.title('Air Quality vs Flu Rate (<18)')
    plt.grid(True)
    plt.savefig('pm25_vs_flu_lt18.png') 
    plt.close()

    plt.figure(figsize=(12,6))
    states = df['state_abbr'].unique()
    for state in states:
        state_df = df[df['state_abbr'] == state].sort_values('year_week')
        plt.plot(state_df['year_week'], state_df['rate_age_lt18'], label=state)

    plt.xlabel('Year Week')
    plt.ylabel('Flu Rate (<18)')
    plt.title('Flu Rate (<18) by State Over Time')
    plt.legend(bbox_to_anchor=(1.05, 1), loc='upper left')
    plt.grid(True)
    plt.savefig('flu_lt18_by_state.png')  
    plt.close()

    print("Plots saved as PNG files in your Cloud9 workspace.")

<p align="left">
<img src="Képernyőfotó 2026-01-04 - 14.42.56.png" alt="S3 Folders" width="700">
</p>

<p align="left">
<img src="Képernyőfotó 2026-01-04 - 14.43.01.png" alt="S3 Folders" width="700">
</p>

<span style="color:red; font-weight:bold;">The scatter plot shows the relationship between pm_2.5 and flu rates in children. The points are widely scattered with no clear trend, confirming a weak linear correlation. Most pm_2.5 values fall between 11 and 13. Although some flu rates reach 6 to 8 in this range, many low flu rates (1 to 2) also occur at the same pollution levels, indicating that other factors, such as season or temperature, likely have a stronger influence.


<span style="color:red; font-weight:bold;">The time-series line graph tracks flu rates for children across several states over five weeks (20201 to 20205). Georgia started with the highest flu rate, near 7.5, but steadily declined, while Connecticut and Utah experienced sharp increases between weeks 2 and 4. These differences show that flu outbreaks vary by state and are influenced more by local conditions and transmission than by a single nationwide environmental factor. While it is also great to see Georgia is always above the other states, that could indicate problems with healthcare in that state, anomaly, disfunction.</span>
