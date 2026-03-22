# Does Wealth Affect How Long the Lights Stay Off?
### By Shriya Konda

Does wealth affect how long the lights stay off? This project explores whether 
the socioeconomic status of a state influences power outage duration across the 
U.S. from 2000 to 2016.

---

## Introduction

This project looks at a dataset of major power outages across the U.S. from 
January 2000 to July 2016. These are not your average outages — they are 
defined by the Department of Energy as events that impacted at least 50,000 
customers or caused an unplanned energy demand loss of at least 300 MegaWatts. 
The data comes from Purdue University's Laboratory for Advancing Sustainable 
Critical Infrastructure.

The dataset has 1534 rows, each one representing a single major outage, and 56 
columns covering everything from the cause of the outage to the economic and 
geographic characteristics of the affected state.

Growing up, I visited my grandparents in a village that dealt with frequent 
power outages. I always wondered whether wealthier areas were able to recover 
faster. That question drives this project: **Do states with higher per capita 
GDP experience shorter power outages than states with lower per capita GDP?** 
This matters because outages affect public health, local economies, and daily 
life. If wealthier states consistently recover faster, that points to real 
inequities in how energy infrastructure is funded and maintained across the 
country.

The columns most relevant to this question are:

| Column | Description |
| --- | --- |
| `OUTAGE.DURATION` | Duration of the outage in minutes |
| `PC.REALGSP.STATE` | Per capita real gross state product (USD) |
| `CAUSE.CATEGORY` | Category of the cause of the outage |
| `CLIMATE.REGION` | U.S. climate region of the affected state |
| `ANOMALY.LEVEL` | Oceanic El Nino/La Nina index at time of outage |
| `POPDEN_URBAN` | Population density of urban areas (persons per sq mile) |
| `MONTH` | Month the outage occurred |
| `U.S._STATE` | State where the outage occurred |

---

## Data Cleaning and Exploratory Data Analysis

### Cleaning

Before doing any analysis, I had to clean up the raw data quite a bit. Here 
is what I did and why:

First, I dropped the first row of the raw file since it contained units rather 
than actual data, and removed the `variables` and `OBS` columns which were 
just metadata leftover from the original Excel formatting.

A lot of columns that should have been numeric were read in as strings because 
of how the Excel file was structured. I converted columns like 
`OUTAGE.DURATION`, `PC.REALGSP.STATE`, `DEMAND.LOSS.MW`, 
`CUSTOMERS.AFFECTED`, `ANOMALY.LEVEL`, and others to numeric values using 
`pd.to_numeric`, which also replaced any unparseable entries with `NaN` 
automatically.

I also combined the separate date and time columns for outage start and 
restoration into single Timestamp columns called `OUTAGE.START` and 
`OUTAGE.RESTORATION`, then dropped the originals since all the information 
was captured in the new columns.

Finally, I created a `HIGH.GDP` binary column by comparing each state's per 
capita GDP to the median across all states. This is used throughout the 
hypothesis test and fairness analysis to split states into high and low GDP 
groups.

Here are the first few rows of the cleaned DataFrame:

| U.S._STATE | CAUSE.CATEGORY | OUTAGE.DURATION | PC.REALGSP.STATE | CLIMATE.REGION |
| --- | --- | --- | --- | --- |
| Minnesota | severe weather | 3060.0 | 51268.0 | East North Central |
| Minnesota | intentional attack | 1.0 | 53499.0 | East North Central |
| Minnesota | severe weather | 3000.0 | 50447.0 | East North Central |
| Minnesota | severe weather | 2550.0 | 51598.0 | East North Central |
| Minnesota | severe weather | 1740.0 | 54431.0 | East North Central |

### Univariate Analysis
