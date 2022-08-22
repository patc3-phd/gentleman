
<!-- README.md is generated from README.Rmd. Please edit that file -->

# gentleman <img src="man/figures/logo.svg" style="float:right; height:200px;" />

<!-- badges: start -->
<!-- badges: end -->

The **gentleman** package provides a suite of modules to help with data
preparation, descriptive statistics, modeling, and publication
formatting. The Descriptives module is extensible to accommodate custom
functions to generate tables and p-values (or other useful statistic to
compare groups). The Models module can be used to generate lavaan model
strings for mediation and cross-lagged models.

## Installation

You can install gentleman from the public repository on GitHub with:

``` r
# install.packages("devtools")
devtools::install_github("patc3-phd/gentleman")
```

## Using gentleman

Once installed, you can use the package by loading it into your R
session.

``` r
library(gentleman)
```

## Modules

The package is composed of several modules that accomplish related tasks
(see the [reference
page](https://d22consulting.com/gentleman/reference/index.html) for a
complete list of functions):

-   Data preparation
-   Descriptive statistics
-   Models & clustering
-   Machine learning
-   Publications
-   Logger
-   Helpers

### 1. Data Preparation

With the data preparation module, you can clean and manipulate
data.frames, and recode variables and modify factors. For example:

``` r
# Remove non-ASCII characters and variables with too many missing values,
# cast character vars as factors, and remove factors with too many categories
df <- df |> 
  remove_non_ascii_from_df() |> 
  remove_vars_with_too_many_missing(pmissing=.80) |> 
  cast("character", "factors") |> 
  remove_factors_with_too_many_levels(maxlevels=20)

# Recode 'Nationality' using a lookup table in an Excel file
df$Nationality <- df$Nationality |> 
  recode_using_excel_map("map.xlsx", sheet="Nationality")
```

### 2. Descriptive Statistics

The package is equipped with an extensive and customizable module for
descriptive statistics. In particular, you can generate
publication-ready tables of descriptive statistics for numeric and
categorical variables, optionally with added columns for group-specific
statistics and group comparison.

For example:

``` r
# Get a table of descriptive statistics for numeric variables
# and test for group differences by treatment group
df |> get_desc_table(
  vars=c("age", "score"),
  tbl_fn=tbl_fn_num(),
  group="TreatmentGroup",
  ana_fn=ana_fn_aov()
)

# Get a table of descriptive statistics for factor variables
# and test for group differences by treatment group
df |> get_desc_table(
  vars=c("Nationality", "Diagnostic"),
  tbl_fn=tbl_fn_fac(),
  group="TreatmentGroup",
  ana_fn=ana_fn_chisq()
)

# Get all variables that significantly vary by groups
df |> get_sig_differences_between_groups(group="TreatmentGroup")

# Plot densities by group
df |> plot_density_by_groups(group="TreatmentGroup")
```

The `get_desc_table()` function is the workhorse of the module, which
takes as arguments:

-   **`tbl_fn`**: A function that returns a table of statistics for
    given variables; the package comes with `tbl_fn_num()` for numeric
    variables, and `tbl_fn_fac()` for categorical (factor) variables
-   **`ana_fn`**: A function that returns a table of test statistics
    (typically *p*-values) to compare groups on a set of variables; the
    package comes with `ana_fn_aov()` and `ana_fn_rm_aov()` for
    (repeated) numeric variables (*p*-values from ANOVA), and
    `ana_fn_chisq()` for categorical (factor) variables (*p*-values from
    Chi-square tests)

You can develop custom `tbl_fn` and `ana_fn` functions as you wish and
pass them as arguments to `get_desc_table()`, which will take care of
the rest.

### 3. Models & Clustering

The models module can generate `lavaan` syntax to estimate complex
mediation and cross-lagged models:

``` r
library(lavaan)

# Mediation model with one x, one mediator, and one y
get_mediation_model("x1", "x2", "x3") |>
  sem(df) |>
  summary()

# Cross-lagged model with 3 different variables assessed at 3 occasions each
vars <- list(
  c("x1", "x2", "x3"),
  c("y1", "y2", "y3"),
  c("z1", "z2", "z3")
)
get_crosslagged_model(vars, random_intercepts=FALSE) |>
  sem(df) |>
  summary()
```

The module can also be used to decompose 2-way interactions in different
types of models (like `lm` and `lmer`):

``` r
fit <- lm(SelfEsteem ~ Gender*AttitudeTowardsSchool, df)
fit |> decompose_interaction(x1="Gender", x2="AttitudeTowardsSchool")
```

Clustering can also be performed easily with mixed types of variables
and missing data:

``` r
# Generate clusters with variable selection using an optimal number of clusters
# and plot densities of variables on which clusters significantly vary
df |>
  add_cluster_assignment(k=NULL, maxit=10, return_df_cluster_instead=TRUE) |>
  plot_density_by_groups(group="Cluster")
```

### 4. Machine Learning

The machine learning module uses the library **h2o** to generate and
process an AutoML model by adding predictions to a test set and
retrieving variable importances. The module also comes with a pipeline
built-in, which takes in a data.frame, splits it into training and test
sets, prepares them for h2o, and requests an AutoML model and obtains
predictions and variable importances.

For example:

``` r
# AutoML pipeline with default configuration (random forest),
# 70%-30% train-test split, scaled numeric features,
# confidence levels for predictions, and variable importances
pip <- df |> run_automl_pipeline(target="y1",
                                 prop_train=.7,
                                 scale_numeric_features=TRUE,
                                 add_confidence_level=TRUE,
                                 plot_and_print_variable_importances=TRUE)
```

### 5. Publications

With the publications module, you can generate publication-ready tables
for different types of models that you can copy-paste, for example into
Excel. Effects are formatted with 2 decimals and no leading zero, and
significance is indicated with common standards using asterisks.
Currently, publication tables are available for `lavaan` models and
models that can be summarized using `broom` (like `lm` and `glm`).

``` r
# Generate a publication table for two regressions using lavaan 
# and full-information maximum likelihood estimation
library(lavaan)
fit_lavaan <- list()
fit_lavaan$y1 <- sem("y1 ~ x1+x2", df, missing="ml.x")
fit_lavaan$y2 <- sem("y2 ~ x1+x2", df, missing="ml.x")
pub_lavaan <- fit_lavaan |> make_pub_table_from_lavaan_models()

# Generate a publication table for two regressions with OLS estimation 
# and listwise deletion of missing data
library(broom)
fit_lm <- list()
fit_lm$y1 <- lm(y1 ~ x1+x2, df) |> tidy()
fit_lm$y2 <- lm(y2 ~ x1+x2, df) |> tidy()
pub_lm <- fit_lm |> make_pub_table_from_broom_tidy()
```

You can also compare significant effects between publication tables:

``` r
# Sensitivity analysis: effect of missing data
pub_lm |> compare_sig_effects_in_two_pub_tables(pub_lavaan)
```

### 6. Logger

The logger module offers a simple way to log console output to a text
file on disk. All logs are saved in a given directory in a folder called
`gentleman_logs`, and the file name can be optionally specified (by
default the file name will be the current date and time). Logs can be
updated by passing a logger object. The module can also be used to
retrieve information about the latest commit in a git repository.

``` r
# Simple one-time log
logtext(expr={
  print("model without covariates:")
  lm(y1 ~ x1, df) |> summary() |> print()
  
  print("model with covariates")
  lm(y1 ~ x1 + x2 + x3, df) |> summary() |> print()
})

# Create a logger with git information and update log
logger <- get_logger(fname="models with and without covariates", add_git_info=TRUE)
logger |> logtext(expr={
  print("model without covariates:")
  lm(y1 ~ x1, df) |> summary() |> print()
})
logger |> logtext(expr={
  print("model with covariates")
  lm(y1 ~ x1 + x2 + x3, df) |> summary() |> print()
})
```

### 7. Helpers

Other miscellaneous helper functions are available. Some are aliases and
wrappers (like `view()`, `copy()` and `copy2()`), others are utility
functions that can fasten development (for example, you can use
`replace_in_nested_list()` to find and replace values in a complex list
of vectors with a nested structure while preserving the structure).

See the [complete list of
functions](https://d22consulting.com/gentleman/reference/index.html) for
more details.
