# Recipe Complexity and Ratings: What Makes a Recipe Successful?

**By:** Uddhav Panchavati, Ryan Namdar

---

## Introduction

This project uses the **Recipes and Ratings** dataset from Food.com, which contains roughly 83,000 unique recipes and over 731,000 user reviews. Our central question is:

**Does recipe complexity -- measured by number of steps and ingredients -- affect average user ratings?**

When someone posts a recipe with 20+ steps, does that scare people off and lead to lower ratings? Or do more involved recipes produce better dishes that earn higher praise? This matters to anyone who writes or follows recipes online, from home cooks deciding what to try tonight to food bloggers figuring out how detailed their instructions should be.

After cleaning and merging the two source files, our dataset has **83,194 rows**. The columns most relevant to our question are:

| Column | Description |
|---|---|
| `n_steps` | Number of preparation steps in the recipe |
| `n_ingredients` | Number of ingredients used |
| `minutes` | Estimated preparation time in minutes |
| `calories` | Caloric content, extracted from the `nutrition` column |
| `avg_rating` | Average star rating (1-5) across all user reviews for that recipe |

---

## Data Cleaning and Exploratory Data Analysis

We took the following cleaning steps before any analysis:

1. **Replaced 0 ratings with NaN.** Food.com stores "no rating given" as 0, which is not a real 1-5 star rating. Leaving these in would drag down averages for recipes that simply had some reviewers who did not leave a star rating.
2. **Computed average rating per recipe** by grouping all reviews by recipe ID and taking the mean of non-missing ratings.
3. **Left-joined** the average ratings back onto the recipes table so every recipe is kept, even those with no reviews (which get NaN for `avg_rating`).
4. **Parsed the `nutrition` column** from a string-encoded list into separate numeric columns: calories, total fat, sugar, sodium, protein, saturated fat, and carbohydrates.
5. **Parsed the `ingredients` column** from a string to a list and counted the number of ingredients.
6. **Converted `submitted`** to a proper datetime type.
7. **Dropped recipes with prep times over 1,440 minutes** (24 hours), since these are almost certainly data entry errors.

Here are the first few rows of the cleaned DataFrame:

| name | n_steps | n_ingredients | minutes | calories | avg_rating |
|---|---|---|---|---|---|
| 1 brownies in the world best ever | 10 | 9 | 40 | 138.4 | 4.0 |
| 1 in canada chocolate chip cookies | 12 | 11 | 45 | 595.1 | 5.0 |
| 412 broccoli casserole | 6 | 9 | 40 | 194.8 | 5.0 |
| millionaire pound cake | 7 | 7 | 120 | 878.3 | 5.0 |
| 2000 meatloaf | 17 | 13 | 90 | 267.0 | 5.0 |

### Univariate Analysis

<iframe src="assets/avg-rating-distribution.html" width="800" height="600" frameborder="0"></iframe>

The distribution of average ratings is heavily left-skewed, with most recipes sitting between 4 and 5 stars. This reflects a strong positivity bias on Food.com -- people tend to rate recipes they liked, and recipes they disliked often go unreviewed.

<iframe src="assets/n-steps-distribution.html" width="800" height="600" frameborder="0"></iframe>

The number of steps per recipe is right-skewed. Most recipes have fewer than 15 steps, with a long tail stretching out past 40. Simple recipes dominate the platform.

### Bivariate Analysis

<iframe src="assets/rating-by-steps.html" width="800" height="600" frameborder="0"></iframe>

When we bin recipes by step count and look at the distribution of ratings within each bin, the medians and quartiles are remarkably similar across all groups. There is no obvious visual trend suggesting that more complex recipes receive lower (or higher) ratings.

<iframe src="assets/rating-vs-ingredients.html" width="800" height="600" frameborder="0"></iframe>

A scatter plot of average rating against number of ingredients (with a LOWESS trendline) shows an essentially flat relationship. Ingredient count does not appear to predict rating in any meaningful way.

### Interesting Aggregates

The table below groups recipes by step count and shows average statistics for each bin:

| step_bin | num_recipes | mean_rating | mean_calories | mean_n_ingredients | mean_minutes |
|---|---|---|---|---|---|
| 1-5 | 18466 | 4.64 | 327.67 | 6.94 | 44.74 |
| 6-10 | 31744 | 4.62 | 400.60 | 8.79 | 58.57 |
| 11-15 | 18248 | 4.62 | 463.18 | 10.33 | 66.75 |
| 16-20 | 7338 | 4.64 | 536.80 | 11.52 | 79.03 |
| 20+ | 4835 | 4.65 | 645.79 | 12.84 | 103.19 |

Mean ratings are nearly identical across all complexity levels (ranging from 4.62 to 4.65). Meanwhile, calories, ingredient counts, and prep times all increase steadily with step count, confirming that these complexity measures track together. The key takeaway is that complexity is associated with richer, more time-consuming recipes, but not with meaningfully different ratings.

---

## Assessment of Missingness

### NMAR Analysis

We believe `avg_rating` is NMAR (Not Missing At Random). Recipes without any reviews get a missing average rating, and the reason they have no reviews is likely tied to the rating they would receive. Recipes that look unappealing or have poor descriptions attract fewer visitors, meaning fewer people cook them and even fewer leave a rating. The missingness depends on the unobserved value itself -- a hallmark of NMAR.

If we had access to page view or impression data for each recipe, we could condition on how many people actually saw the recipe. That would let us separate "nobody saw it" from "people saw it but chose not to rate it," potentially turning the missingness into MAR.

### Missingness Dependency

We ran two permutation tests to check whether the missingness of `avg_rating` depends on other observed columns, using the absolute difference in group means as our test statistic and a significance level of 0.05.

**Test 1: Missingness of `avg_rating` vs. `n_steps`**

- Null hypothesis: The distribution of `n_steps` is the same whether `avg_rating` is missing or not.
- Alternative hypothesis: The distributions differ.
- Mean `n_steps` when rating is missing: 11.47
- Mean `n_steps` when rating is observed: 10.03
- Observed absolute difference in means: 1.44
- P-value: < 0.001
- Result: We rejected the null hypothesis. The missingness of `avg_rating` does depend on `n_steps`. Recipes with missing ratings tend to have more steps on average, which makes sense -- more complex recipes may attract fewer reviewers.

<iframe src="assets/missingness-steps.html" width="800" height="600" frameborder="0"></iframe>

The plot above shows the empirical distribution of the test statistic under the null, with the observed statistic marked by the red dashed line. The observed difference falls well outside the permutation distribution, confirming the dependency.

**Test 2: Missingness of `avg_rating` vs. `calories`**

- Null hypothesis: The distribution of `calories` is the same whether `avg_rating` is missing or not.
- Alternative hypothesis: The distributions differ.
- Mean calories when rating is missing: 510.01
- Mean calories when rating is observed: 425.16
- Observed absolute difference in means: 84.85
- P-value: < 0.001
- Result: We rejected the null hypothesis. The missingness of `avg_rating` also depends on `calories`. Recipes with missing ratings tend to be higher in calories, suggesting that more extreme or indulgent recipes are less likely to be reviewed.

<iframe src="assets/missingness-calories.html" width="800" height="600" frameborder="0"></iframe>

The observed calorie difference also falls far outside the permutation distribution, confirming that the missingness of ratings depends on calorie count as well.

**Test 3: Missingness of `avg_rating` vs. `sugar` (not dependent)**

- Null hypothesis: The distribution of `sugar` is the same whether `avg_rating` is missing or not.
- Alternative hypothesis: The distributions differ.
- Test statistic: Absolute difference in group means.
- Significance level: 0.05
- Result: We fail to reject the null hypothesis. The missingness of `avg_rating` does not depend on `sugar`. Sugar content is a nutritional attribute that does not drive whether a recipe attracts reviewers, so it is unsurprising that missing and observed recipes have similar sugar distributions.

<iframe src="assets/missingness-sugar.html" width="800" height="600" frameborder="0"></iframe>

---

## Hypothesis Testing

We tested whether recipe complexity (measured by step count) affects average ratings using a permutation test.

- **Null hypothesis:** The average rating for recipes above the median step count is the same as for recipes at or below the median. Any observed difference is due to chance.
- **Alternative hypothesis:** Recipes with more steps have lower average ratings than simpler recipes.
- **Test statistic:** mean(low-step ratings) minus mean(high-step ratings). A positive value supports the alternative.
- **Significance level:** 0.05

We chose a permutation test because it makes no distributional assumptions about ratings, which are heavily skewed. The one-sided alternative reflects our initial hypothesis that complexity might hurt ratings.

**Results:**
- Mean rating for low-step recipes (9 steps or fewer): 4.6266
- Mean rating for high-step recipes (more than 9 steps): 4.6234
- Observed difference: 0.0032
- P-value: 0.238

We fail to reject the null hypothesis. The observed difference of 0.003 stars is tiny, and the p-value of 0.238 tells us a difference this large or larger would appear about 24% of the time by random chance alone. There is not enough evidence to conclude that complex recipes receive lower ratings.

---

## Framing a Prediction Problem

**Problem:** Predict the average rating of a recipe based on its characteristics.

This is a **regression** task since `avg_rating` is a continuous variable between 1 and 5.

**Response variable:** `avg_rating` -- chosen because it directly measures how well a recipe is received, tying back to our central question about what makes a recipe successful.

**Evaluation metric:** RMSE (Root Mean Squared Error). We chose RMSE over MAE because it penalizes large prediction errors more heavily. Predicting a 1-star recipe as 5 stars is much worse than being off by a fraction of a star, and RMSE captures that. It is also in the same units as the response variable (stars), making it straightforward to interpret.

**Features used (all known at time of prediction, before any ratings exist):**
- `n_steps` -- number of preparation steps
- `n_ingredients` -- number of ingredients
- `minutes` -- estimated prep time
- `calories` -- caloric content
- `sugar` -- sugar content (% daily value)
- `total_fat` -- fat content (% daily value)

We deliberately excluded any review text, rating counts, or other post-publication data, since those would not be available when a recipe is first posted.

---

## Baseline Model

Our baseline is a **Linear Regression** model with five features:

- **Quantitative features (4):** `n_steps`, `n_ingredients`, `minutes`, `calories` -- passed through without transformation.
- **Nominal feature (1):** `step_bin` (a categorical binning of step count into 1-5, 6-10, 11-15, 16-20, 20+) -- encoded with one-hot encoding, dropping the first category to avoid multicollinearity.

We used an 80/20 train-test split with a fixed random state.

**Performance:**
- Training RMSE: 0.6394
- Test RMSE: 0.6425
- For reference, always predicting the mean rating gives RMSE of 0.6428

This model is not good. Its test RMSE (0.6425) is barely below the naive baseline of always predicting the mean (0.6428), meaning it is hardly learning anything from the features. The near-identical training and test RMSE also suggests the model is underfitting rather than overfitting. The linear relationship between these features and ratings is essentially nonexistent, which lines up with what we saw in our exploratory analysis.

---

## Final Model

We added two engineered features and switched to a non-linear model:

**New features:**

1. **`log_calories`:** A log transform of calories. The raw calorie distribution has extreme outliers (up to ~45,000 calories), and the difference between a 200-calorie and 400-calorie recipe is much more meaningful than the difference between a 10,000-calorie and 10,200-calorie one. The log transform compresses these outliers and better captures the diminishing returns of calorie increases.

2. **`protein_to_fat_ratio`:** Protein divided by (total fat + 1). This ratio captures how "balanced" a recipe is nutritionally. Health-conscious users may rate nutritionally balanced recipes differently than indulgent ones, and neither protein nor fat alone captures this relationship.

**Additional transformations:**
- `StandardScaler` on `n_steps` and `n_ingredients` to normalize their scales
- `QuantileTransformer` on `minutes` to handle its heavy right skew

**Model:** `RandomForestRegressor` with 100 trees. We chose a Random Forest because the baseline linear model showed that the relationship between features and ratings is not linear. Random Forests can capture non-linear patterns and interactions between features without requiring us to specify them manually.

**Hyperparameter tuning:** We used 5-fold cross-validation with `GridSearchCV` to tune `max_depth` over the values [5, 10, 15, 20, 30, None]. Tree depth directly controls the bias-variance tradeoff -- too shallow and the model underfits, too deep and it memorizes training noise.

**Performance:**
- Best `max_depth`: 5
- Training RMSE: 0.6360
- Test RMSE: 0.6413
- Baseline model test RMSE: 0.6425
- Improvement: 0.0012

The final model's test RMSE of 0.6413 is an improvement over the baseline's 0.6425. The improvement is small in absolute terms, which reflects the fundamental difficulty of predicting ratings from recipe metadata alone -- as our earlier analysis showed, ratings barely vary with complexity. That said, the Random Forest with engineered features and a shallow max depth of 5 does consistently outperform the linear baseline, and the gap between training and test RMSE (0.6360 vs. 0.6413) shows the model generalizes reasonably without heavy overfitting.

---

## Fairness Analysis

We tested whether our final model performs equally well for low-calorie and high-calorie recipes, split at the median calorie count in the test set.

- **Group X:** Low-calorie recipes (calories at or below the median)
- **Group Y:** High-calorie recipes (calories above the median)
- **Evaluation metric:** RMSE
- **Null hypothesis:** The model is fair. Its RMSE for low-calorie and high-calorie recipes is roughly the same, and any observed difference is due to random chance.
- **Alternative hypothesis:** The model is unfair. Its RMSE for high-calorie recipes is higher than for low-calorie recipes.
- **Test statistic:** RMSE(high-calorie) minus RMSE(low-calorie). A positive value supports the alternative.
- **Significance level:** 0.05

We ran a permutation test with 1,000 iterations, shuffling the calorie group labels each time and recomputing the RMSE difference.

**Results:**
- RMSE for low-calorie recipes: 0.6473
- RMSE for high-calorie recipes: 0.6353
- Observed difference (high minus low): -0.0120
- P-value: 0.782

The observed difference is actually negative, meaning the model performs slightly better on high-calorie recipes, not worse. With a p-value of 0.782, this difference is well within the range we would expect from random chance. We fail to reject the null hypothesis at the 0.05 significance level. There is not sufficient evidence to conclude that the model performs unfairly across calorie groups.

<iframe src="assets/fairness-analysis.html" width="800" height="600" frameborder="0"></iframe>
