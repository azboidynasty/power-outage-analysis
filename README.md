# No Power For How Long???

Andrew Zhou

andrewjz@umich.edu

## Introduction

**Ever wondered how long a power outage will last based on where you are and what time of year it is?**

A dataset [here](https://engineering.purdue.edu/LASCI/research-data/outages), consisting of major power outages across the United States can help us answer this question. Besides major outages, this data contains information on geographical location of the outages, regional climatic information, land-use characteristics, electricity consumption patterns and economic characteristics of the states affected by the outages. 

The original raw dataframe is 1534 rows (outages) x 56 columns.

However, for the purposes of answering the question above, I will focus on:

- 'MONTH', the month in which the outage occurred
- 'POSTAL.CODE', the state in which the outage occurred, represented as a two letter postal code (e.g. CA, TX, NY)
- 'OUTAGE.DURATION', how long the outage lasted, in minutes

If we can succesfully answer our question, this will help energy providers optimally allocate resources to restore power as quickly as possible depending on the time of year and location.

## Data Cleaning and Exploratory Data Analysis

### Data Cleaning

1. First, keep the columns relevant to the question, namely **MONTH, POSTAL.CODE,** and **OUTAGE.DURATION**, and drop the rest of the columns.

2. A power outage that lasts 0 minutes shouldn't really count as an outage. Also, an outage that lasts 1 minute isn't something most people would be too concerned about. Let's drop any outages that last 0 or 1 minutes.

3. It turns out there are no missing values for state postal code. Let's group by state, which will let us see for each state, how many power outages occurred, the mean duration, and also help us analyze missingness.

| POSTAL.CODE   |   missing_months |   mean_outage_duration |   count_power_outage |   missing_outage_duration |
|:--------------|-----------------:|-----------------------:|---------------------:|--------------------------:|
| AK            |                1 |                 nan    |                    1 |                         1 |
| AL            |                1 |                1152.8  |                    6 |                         1 |
| AR            |                0 |                1577.42 |                   24 |                         0 |
| AZ            |                0 |                5419.95 |                   24 |                         3 |
| CA            |                0 |                1736.46 |                  202 |                        12 |

4. There are 9 missing values for month, but this isn't too many compared to a total of over 1000 outages. Additionally, they appear to be missing at random. We can drop these outages.

5. There are also missing values for outage duration. 

    - For Alaska, there's only one outage in this dataset, and it has a missing value for outage duration. Unfortunately, imputation might not be the best solution here, since we don't have a reliable method or additional information about power outages in Alaska to determine the best value for imputation.

    - For California, notice that there are significantly more missing values for outage duration. Fortunately, this should be expected, as California is a large and heavily populated state and hence has more power outages. The proportion of missing values for outage duration to the total number of power outages doesn't seem to fluctuate too much from state to state. We can drop these too.

    - Since only a relatively small percentage of values are missing, and at random, we can drop such outages.

6. Here is a preview of the cleaned dataframe:

|   MONTH | POSTAL.CODE   |   OUTAGE.DURATION |
|--------:|:--------------|------------------:|
|       7 | MN            |              3060 |
|      10 | MN            |              3000 |
|       6 | MN            |              2550 |
|       7 | MN            |              1740 |
|      11 | MN            |              1860 |

### Exploratory Data Analysis

<iframe
  src="assets/outage_duration_histogram.html"
  width="800"
  height="600"
  frameborder="0"
></iframe>

Above is the distribution of outage durations across the country. It appears that most power outages last a shorter amount of time, with a few more signifcant ones sprinkled in. It's important to keep in mind that this distribution is asymmetrical when we work with this later.

<iframe
  src="assets/outage_duration_vs_month.html"
  width="800"
  height="600"
  frameborder="0"
></iframe>

Across the United States, with the exception of a few outliers here and there, the distributions of outages durations by month appear to be similar.

Outliers have a bad rap, and most of the time, our goal is to remove them. In the context of our dataset, although they are relatively uncommon, they inevitably happen (e.g. natural disaster, severe weather, other anomalies), and we must be prepared for them and take them into account.

Now, let's take a look at what happens when we examine one state in particular.

<iframe
  src="assets/outage_duration_vs_month_florida.html"
  width="800"
  height="600"
  frameborder="0"
></iframe>


Florida is infamous for being struck by significantly more hurricanes than most other states. If we analyze the distributions of outage durations by month in Florida, we can see that August, September, and October, the peak of hurricane season, have the longest power outages, where storms can knock out power for several days at a time.

The United States is such a large and geographically diverse country, which makes it basically impossible to generalize outage durations for the entire country. This specific plot tells us that location matters when predicting outage duration.

## Framing a Prediction Problem

**For a month of the year and a location inside the country, how long should I expect a power outage there and then to last?**

This is a regression problem, as we are predicting outage duration, a continuously distributed quantitative variable.

Outage duration is a natural response variable, and predicting it is helpful in many ways. Most importantly, it could help energy providers improve resource allocation to restore power as quickly as possible depending on time and location.

Mean absolute error is likely the best metric for model performance evaluation. MAE returns the average error in absolute terms, in the same units as outage duration. Additionally, since we elected not to drop outliers, MAE is also resistant to outliers, meaning that outliers won't disproportionately affect our evaluation metrics.

Mean squared error, unfortunately, is sensitive to outliers. R^2^, while great for linear models, can be misleading for non-linear models, and is not as straightforward to interpret as MAE.

## Baseline Model

Based on a month and state postal code, my model will predict outage duration.

We have an ordinal feature, *month*, conveniently already represented numerically for us. We also have a nominal feature, *state postal code*, which we can one hot encode. Our response variable, *outage duration*, is a quantitative variable, as mentioned earlier.

For a baseline model, I fit a least squares linear regression model.

Unfortunately, the model's performance turned out to be quite poor.

Our MAE turns out to be nearly 3000 minutes, about 50 hours. Our relative MAE, the ratio of the MAE to the mean of the response variable, is over 1. Additionally, R^2^ < 1. All of these metrics indicate poor performance.

In addition to a model that performs poorly on this data, the cross-validated MAE was very large, indicating that furthermore, our baseline model does not generalize well to unseen data, presumably because it also fails to perform well on existing data.

## Final Model

To try to improve our model, this time I used *RandomForestRegressor*, an algorithm based on decision trees and uses results from multiple models to create more accurate and stable predictions, which is good for handling different types of features as well as different types of relationships, including non-linear relationships, as well as being resistant to outliers.

Remember that the distribution of outage durations is skewed right. We'd ideally want a symmetric distribution, so let's apply a log transformation.

Note that month is represented as whole numbers from 1 to 12. Although random forests handle non-scaled features quite well, and it likely won't make a siginificant difference, transforming this to a normal distribution shouldn't hurt.

It's also likely that outage durations are more influenced by season rather than month. For instance, outages in January and February might be similar in duration due to similarities in cause like weather factors. We can create a new feature for seasons based on month, which can slightly simplify the encoding process.

Using GridSearchCV to tune hyperparameters in the ranges:

```
'model__n_estimators': [50, 100, 200],
'model__max_depth': [10, 20, None],
'model__min_samples_split': [2, 5, 10]
```

The best hyperparameters turned out to be:

```
'model__max_depth': 20,
'model__min_samples_split': 10,
'model__n_estimators': 200
```

Our new MAE, although still not ideal, is a slight improvement, decreasing to around 2800 minutes, or around 46.67 hours.

In the original feature space, we have a slightly improved, but not excellent, relative MAE, being able to just about decrease it to under 100%.

On the other hand, in the transformed space, our relative MAE is at around 20%, which indicates very good performance in the transformed space. This is expected, since after feature engineering, we can more easily work with standardized and symmetric distributions.

Our new cross-validated MAE is fortunately much lower, so hopefully, our new model generalizes well to unseen data.