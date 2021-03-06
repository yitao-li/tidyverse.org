---
title: hardhat 0.1.0
author: Davis Vaughan
date: '2019-12-16'
slug: hardhat-0-1-0
description: 
  hardhat 0.1.0 is now available on CRAN. It provides tools for developing new
  modeling packages, with a focus around preprocessing, predicting, and
  validating input.
categories:
  - package
tags:
  - tidymodels
  - hardhat
photo: 
  url: https://unsplash.com/photos/YSxcf6C_SEg
  author: Silvia Brazzoduro
---



We're excited to announce that the first version of [hardhat](https://tidymodels.github.io/hardhat/) is now on CRAN. hardhat is a developer-focused package with the goal of making it easier to create new modeling packages, while simultaneously promoting good R modeling package standards. To accomplish this, hardhat provides tooling around preprocessing, predicting, and validating user input, along with a way to set up the structure of a new modeling package with a single function call.


```r
library(modeldata)
library(tibble)
library(hardhat)
library(recipes)

data("biomass")
biomass <- as_tibble(biomass)
```

## Setup

One exciting feature included with hardhat is `create_modeling_package()`. Built on top of `usethis::create_package()`, this allows you to quickly set up a new modeling package with pre-generated infrastructure in place for an S3 generic to go with your user-facing modeling function. It also includes a `predict()` method and other best practices outlined further in [Conventions for R Modeling Packages](https://tidymodels.github.io/model-implementation-principles/). If you've never created a modeling package before, this is a great place to start so you can focus more on the implementation rather than the details around package setup.

## Preprocessing

When building a model, there are often preprocessing steps that you perform on the training set before fitting. Take this `biomass` dataset for example. The goal is to predict the `HHV` for each sample, an acronym for the Higher Heating Value, defined as the amount of heat released by an object during combustion. To do this, you might use the numeric columns containing the amounts of different atomic elements that make up each sample.


```r
training <- biomass[biomass$dataset == "Training",]
testing <- biomass[biomass$dataset == "Testing",]

training
#> # A tibble: 456 x 8
#>    sample                 dataset  carbon hydrogen oxygen nitrogen sulfur   HHV
#>    <chr>                  <chr>     <dbl>    <dbl>  <dbl>    <dbl>  <dbl> <dbl>
#>  1 Akhrot Shell           Training   49.8     5.64   42.9     0.41   0     20.0
#>  2 Alabama Oak Wood Waste Training   49.5     5.7    41.3     0.2    0     19.2
#>  3 Alder                  Training   47.8     5.8    46.2     0.11   0.02  18.3
#>  4 Alfalfa                Training   45.1     4.97   35.6     3.3    0.16  18.2
#>  5 Alfalfa Seed Straw     Training   46.8     5.4    40.7     1      0.02  18.4
#>  6 Alfalfa Stalks         Training   45.4     5.75   40.2     2.04   0.1   18.5
#>  7 Alfalfa Stems          Training   47.2     5.99   38.2     2.68   0.2   18.7
#>  8 Alfalfa Straw          Training   45.7     5.7    39.7     1.7    0.2   18.3
#>  9 Almond                 Training   48.8     5.5    40.9     0.8    0     18.6
#> 10 Almond Hull            Training   47.1     5.9    40       1.2    0.1   18.9
#> # … with 446 more rows
```

Depending on the model you choose, you might need to center and scale your data before passing it along to the fitting function. There are two main ways you might do this: a formula, or a recipe. As a modeling package developer, ideally you'd support both in your user-facing modeling function, like so:


```r
my_model <- function(x, ...) {
  UseMethod("my_model")
}

my_model.formula <- function(formula, data, ...) {
  
}

my_model.recipe <- function(x, data, ...) {
  
}
```

Unfortunately, each have their own nuances and tricks to be aware of, which you probably don't want to spend too much time thinking about. Ideally, you'd be able to focus on your package's implementation, and easily be able to support a number of different user input methods. This is where hardhat can help. `hardhat::mold()` is a preprocessing function that knows how to preprocess formulas, prep recipes, and deal with the more basic XY input (two data frames, one holding predictors and one holding outcomes). The best part is that the output from `mold()` is standardized across all 3 preprocessing methods, so you always know what data structures you'll be getting back.


```r
rec <- recipe(HHV ~ carbon, training) %>%
  step_normalize(carbon)

processed_formula <- mold(HHV ~ scale(carbon), training)
processed_recipe <- mold(rec, training)

names(processed_formula)
#> [1] "predictors" "outcomes"   "blueprint"  "extras"
names(processed_recipe)
#> [1] "predictors" "outcomes"   "blueprint"  "extras"
```

- `predictors` is a data frame of the preprocessed predictors.

- `outcomes` is a data frame of the preprocessed outcomes.

- `blueprint` is the best part of hardhat. It records the preprocessing activities, so that it can replay them on top of new data that needs to be preprocessed at prediction time.

- `extras` is a data frame of any "extra" columns in your data set that aren't considered predictors or outcomes. These might be offsets in a formula, or extra roles from a recipe.


```r
processed_recipe$predictors
#> # A tibble: 456 x 1
#>     carbon
#>      <dbl>
#>  1  0.140 
#>  2  0.110 
#>  3 -0.0513
#>  4 -0.313 
#>  5 -0.153 
#>  6 -0.284 
#>  7 -0.114 
#>  8 -0.255 
#>  9  0.0428
#> 10 -0.120 
#> # … with 446 more rows

processed_recipe$outcomes
#> # A tibble: 456 x 1
#>      HHV
#>    <dbl>
#>  1  20.0
#>  2  19.2
#>  3  18.3
#>  4  18.2
#>  5  18.4
#>  6  18.5
#>  7  18.7
#>  8  18.3
#>  9  18.6
#> 10  18.9
#> # … with 446 more rows
```


Generally you won't call `mold()` interactively, but will, instead, call it from your top-level modeling function as the first step to standardize and validate a user's input.


```r
my_model <- function(x, ...) {
  UseMethod("my_model")
}

my_model.formula <- function(formula, data, ...) {
  processed <- hardhat::mold(formula, data)
  # ... pass on to implementation
}

my_model.recipe <- function(x, data, ...) {
  processed <- hardhat::mold(x, data)
  # ... pass on to implementation
}
```

## Predicting

Once you've used the preprocessed data to fit your model, you'll probably want to make predictions on a test set. To do this, you'll need to reapply any preprocessing that you did on the training set to the test set as well. hardhat makes this easy with `hardhat::forge()`. `forge()` takes a data frame of new predictors, as well as a `blueprint` that was created in the call to `mold()`, and reapplies the correct preprocessing for you. Again, no matter what the original preprocessing method was, the output is consistent and predictable.


```r
forged_formula <- forge(testing, processed_formula$blueprint)
forged_recipe <- forge(testing, processed_recipe$blueprint)

names(forged_formula)
#> [1] "predictors" "outcomes"   "extras"
names(forged_recipe)
#> [1] "predictors" "outcomes"   "extras"

forged_recipe$predictors
#> # A tibble: 80 x 1
#>     carbon
#>      <dbl>
#>  1 -0.193 
#>  2 -0.490 
#>  3 -0.543 
#>  4 -0.188 
#>  5  0.0390
#>  6 -0.390 
#>  7 -0.904 
#>  8 -0.601 
#>  9 -1.84  
#> 10 -1.97  
#> # … with 70 more rows
```

Like `mold()`, `forge()` is not intended for interactive use. Instead, you'll call it from your `predict()` method.


```r
predict.my_model <- function(object, new_data, ...) {
  processed <- hardhat::forge(new_data, object$blueprint)
  # ... pass on to predict() implementation
}
```

`object` here is a model fit of class `"my_model"` that should be the result of a user calling your high level `my_model()` function. To enable `forge()` to work as shown here, you'll need to attach and return the `blueprint` that is created from `mold()` to this model `object`.

`forge()` has powerful data type validation built in. It checks for a number of things including:

- Missing predictors

- Predictors with the correct name, but wrong data type

- Factor predictors with "novel levels"

- Factor predictors with missing levels, which can be recovered automatically

## Learning more

There are 3 key vignettes for hardhat.

- [Creating Modeling Packages With hardhat](https://tidymodels.github.io/hardhat/articles/package.html)

- [Molding data for modeling](https://tidymodels.github.io/hardhat/articles/mold.html)

- [Forging data for predictions](https://tidymodels.github.io/hardhat/articles/forge.html)

There is also a video of Max Kuhn speaking about hardhat at the [XI Jornadas de Usuarios de R conference](https://canal.uned.es/video/5dd25b9f5578f275e407dd88).
