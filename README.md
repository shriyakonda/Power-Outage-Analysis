# Does Wealth Affect How Long the Lights Stay Off?
### By Shriya Konda

Does wealth affect how long the lights stay off? This project explores whether
the socioeconomic status of a state influences power outage duration across the
U.S. from 2000 to 2016.

---

## Introduction

This project looks at a dataset of major power outages across the U.S. from
January 2000 to July 2016. These are not your average outages - they are
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

Before I could do any real analysis, I had to spend some time cleaning up the
raw data. The file came in a weird Excel format with extra header rows, so the
first thing I did was skip those rows when loading it in, drop the units row,
and remove some metadata columns like `variables` and `OBS` that were not
useful for analysis.

Something I noticed pretty quickly was that a lot of numeric columns had been
read in as strings. Things like `OUTAGE.DURATION`, `PC.REALGSP.STATE`,
`DEMAND.LOSS.MW`, and `CUSTOMERS.AFFECTED` all needed to be converted before
I could actually work with them. I used `pd.to_numeric` with `errors='coerce'`
which also conveniently replaced any weird unparseable values with `NaN`.

I also combined the separate date and time columns into proper Timestamp
columns. Instead of having `OUTAGE.START.DATE` and `OUTAGE.START.TIME` as two
separate things, I merged them into a single `OUTAGE.START` column and did the
same for restoration. This made working with time-based features a lot cleaner.

Finally, I created a `HIGH.GDP` column that flags whether a state's per capita
GDP is above or below the median. I use this throughout the hypothesis test and
fairness analysis to compare high and low GDP states.

Here are the first few rows of the cleaned DataFrame:

| U.S._STATE | CAUSE.CATEGORY | OUTAGE.DURATION | PC.REALGSP.STATE | CLIMATE.REGION |
| --- | --- | --- | --- | --- |
| Minnesota | severe weather | 3060.0 | 51268.0 | East North Central |
| Minnesota | intentional attack | 1.0 | 53499.0 | East North Central |
| Minnesota | severe weather | 3000.0 | 50447.0 | East North Central |
| Minnesota | severe weather | 2550.0 | 51598.0 | East North Central |
| Minnesota | severe weather | 1740.0 | 54431.0 | East North Central |

### Univariate Analysis

The first thing I wanted to understand was what the distribution of outage
duration actually looks like. The answer is that it is extremely skewed - most
outages are short, but there are some that last for weeks or even months. I
used a log scale here because without it, those extreme outliers make it
impossible to see what is going on with the majority of the data.

<iframe src="assets/duration_dist.html" width="800" height="500" frameborder="0"></iframe>

I also wanted to see what kinds of outages are most common. Severe weather
dominates by a lot, which makes sense given how much of the U.S. is exposed
to hurricanes, ice storms, and extreme heat. Intentional attacks being second
was honestly a little surprising to me.

<iframe src="assets/cause_counts.html" width="800" height="500" frameborder="0"></iframe>

### Bivariate Analysis

This scatter plot is what got me thinking about GDP and outage duration in the
first place. There is not a clean linear relationship, but you can see that
the really extreme outliers in duration tend to cluster in lower GDP states.
The high GDP states on the right side of the plot seem to have shorter, more
contained outages overall.

<iframe src="assets/gdp_scatter.html" width="800" height="500" frameborder="0"></iframe>

This box plot breaks down outage duration by cause category. The fuel supply
emergency box really stands out - those outages last way longer than anything
else. Intentional attacks on the other hand tend to get resolved pretty
quickly, which makes sense since those are usually targeted and localized.

<iframe src="assets/box_cause.html" width="800" height="500" frameborder="0"></iframe>

### Grouping and Aggregates

This table summarizes the average outage duration and average customers
affected for each cause category. The contrast between fuel supply emergencies
and intentional attacks is striking - one lasts forever but affects almost
nobody, while system operability disruptions are shorter but can knock out
power for hundreds of thousands of people at once.

| CAUSE.CATEGORY                | Avg_Duration | Avg_Customers_Affected | Count |
|:------------------------------|-------------:|-----------------------:|------:|
| fuel supply emergency         |      13484.0 |                    0.1 |    38 |
| severe weather                |       3884.0 |               188574.8 |   744 |
| equipment failure             |       1816.9 |               101935.6 |    55 |
| public appeal                 |       1468.4 |                 7618.8 |    69 |
| system operability disruption |        728.9 |               211066.0 |   123 |
| intentional attack            |        430.0 |                 1790.5 |   403 |
| islanding                     |        200.5 |                 6169.1 |    44 |

---

## Assessment of Missingness

### NMAR Analysis

One column I think is likely MNAR is `DEMAND.LOSS.MW`. My reasoning is that
utilities probably do not bother reporting demand loss when the number is very
small or basically zero. That means the missingness is related to the actual
value itself, which is what makes it MNAR. To check whether it might actually
be MAR, I would want to know which specific utility company reported each
outage. If certain companies just never report demand loss regardless of the
value, then the missingness would depend on the company rather than the value,
which would make it MAR.

### Missingness Dependency

I decided to look at the missingness of `DEMAND.LOSS.MW` since it is missing
for almost half the dataset, which is too significant to ignore.

**Does depend on: `CAUSE.CATEGORY`**

When I looked at the missingness rate of `DEMAND.LOSS.MW` broken down by cause
category, the differences were pretty stark. Public appeal outages are missing
demand loss about 57% of the time, while islanding outages are only missing it
about 15% of the time. I ran a permutation test and got a p-value of 0.0, so I
reject the null hypothesis. The missingness of `DEMAND.LOSS.MW` is clearly
dependent on `CAUSE.CATEGORY`.

The plot below shows the distribution of outage duration when demand loss is
missing versus not missing. The two distributions look pretty different from
each other, which gives more visual evidence that this missingness is not
random.

<iframe src="assets/missingness_dist.html" width="800" height="500" frameborder="0"></iframe>

**Does not depend on: `YEAR`**

I also tested whether the missingness depended on what year the outage
occurred. Here I got a p-value of around 0.17, which is above 0.05, so I fail
to reject the null hypothesis. The missingness rate is pretty consistent across
years, which suggests that whatever is causing the missing values, it is not
related to when the outage happened.

---

## Hypothesis Testing

The main question I wanted to test was whether low-GDP states actually
experience longer outages on average than high-GDP states, or whether any
difference we see is just random chance.

**Null Hypothesis:** The average outage duration in high-GDP states and
low-GDP states is the same. Any observed difference is due to random chance.

**Alternative Hypothesis:** Low-GDP states have a higher average outage
duration than high-GDP states, suggesting wealthier areas recover faster.

**Test Statistic:** Difference in means (low GDP mean minus high GDP mean). I
chose this because my alternative hypothesis is directional and outage duration
is a continuous variable, so a difference in means makes sense here.

**Significance Level:** 0.05

I ran a permutation test with 1000 simulations. The observed difference in
mean outage duration was 942.26 minutes, and the p-value came out to 0.0.
Since that is well below 0.05, I reject the null hypothesis. The data suggests
that low-GDP states do tend to experience longer outages than high-GDP states.
That said, since this is an observational study and not a randomized controlled
trial, I cannot say that lower GDP directly causes longer outages. There could
be other factors at play.

<iframe src="assets/hyp_test.html" width="800" height="500" frameborder="0"></iframe>

---

## Framing a Prediction Problem

For my prediction problem, I am trying to predict `OUTAGE.DURATION`, which is
how long a power outage lasts in minutes. Since this is a continuous numerical
value, this is a regression problem.

I chose `OUTAGE.DURATION` as my response variable because it is the most
direct measure of how badly an outage impacts people. A 10-minute outage is
very different from a 3-day one, and I wanted to understand what factors drive
that difference, especially economic ones like GDP which is what my earlier
analysis focused on.

In terms of what I can actually use to train my model, I had to be careful. At
the time an outage starts, I would realistically know things like the cause,
the state's GDP, the climate region, urban population density, the month, and
the climate anomaly level. I would NOT know `DEMAND.LOSS.MW`,
`CUSTOMERS.AFFECTED`, or `OUTAGE.RESTORATION` since those are only measurable
after the outage is already over, so I left those out.

For my evaluation metric I am using RMSE. I chose this over R-squared because
RMSE is in the same units as my target variable (minutes), which makes it way
easier to interpret. I also prefer it over MAE because it penalizes really
large errors more heavily. If my model massively underestimates a week-long
outage, that is a much bigger deal than being slightly off on a short one.

---

## Baseline Model

My baseline model predicts `OUTAGE.DURATION` using two features in a single
sklearn Pipeline with a Linear Regression model.

The features are:
- `CAUSE.CATEGORY` (nominal) - one-hot encoded since it has no natural ordering
- `PC.REALGSP.STATE` (quantitative) - left as-is since it is already numeric

I chose these two because they are both available at the time of prediction
and are directly tied to my research question about whether wealth affects
outage duration.

The train RMSE was 4915.57 minutes and the test RMSE was 7186.93 minutes. The
gap between training and test performance suggests the model is overfitting a
bit, and the overall high RMSE tells me that a simple linear model with just
two features is not really capturing what drives outage duration. But that is
kind of the point of a baseline - it gives me something concrete to improve on.

---

## Final Model

For my final model I switched to a Random Forest Regressor, which handles
non-linear relationships and interactions between features much better than
linear regression does.

On top of the two features from the baseline, I added:

- `CLIMATE.REGION` (nominal, one-hot encoded) - different climate regions
experience very different weather patterns, which directly affects how severe
outages get. My EDA showed that East North Central had by far the longest
average outage duration, so this felt like an important thing to include.

- `POPDEN_URBAN` (quantile transformed) - how densely populated an area is
affects how quickly repair crews can get to the problem and how much
infrastructure needs to be restored. I used a QuantileTransformer here because
the column is pretty skewed.

- `ANOMALY.LEVEL` (standardized) - unusual climate conditions can prolong
outages beyond what the cause category alone would predict, so I wanted to
capture that.

- `MONTH` (passthrough) - outages in winter and summer tend to be worse
because of harsher conditions and higher energy demand, so seasonality seemed
worth including.

I used GridSearchCV with 5-fold cross validation to tune `n_estimators` and
`max_depth`. The best hyperparameters were `max_depth: 10` and
`n_estimators: 100`. The final model achieved a test RMSE of 7192.9 minutes
compared to the baseline test RMSE of 7186.93 minutes. While the improvement
is modest, the final model is much less overfit - the training RMSE dropped
from 4915.57 to 2482.52, showing the model is generalizing better even if the
test performance is similar.

---

## Fairness Analysis

For my fairness analysis I wanted to check whether my model performs
differently for high-GDP states versus low-GDP states. This connects directly
to the central question of my whole project, so it felt like the natural choice.

**Group X:** Low-GDP states (below median per capita GDP)
**Group Y:** High-GDP states (above median per capita GDP)
**Evaluation Metric:** RMSE
**Test Statistic:** Difference in RMSE (low GDP RMSE minus high GDP RMSE)
**Significance Level:** 0.05

**Null Hypothesis:** The model is fair. Its RMSE for high-GDP and low-GDP
states is roughly the same, and any differences are just due to random chance.

**Alternative Hypothesis:** The model is unfair. Its RMSE for low-GDP states
is higher than its RMSE for high-GDP states.

I ran a permutation test with 1000 trials and got a p-value of 0.05. Since
this is right at our significance level, the result is borderline. The
observed RMSE difference was 6409.97 minutes, meaning the model does perform
noticeably worse for low-GDP states. This makes some sense - low-GDP states
may have more unpredictable outage patterns that are harder for the model to
pin down, possibly because they have less consistent infrastructure or
reporting. While we cannot definitively conclude the model is unfair, this is
worth keeping in mind.

<iframe src="assets/fairness.html" width="800" height="500" frameborder="0"></iframe>
