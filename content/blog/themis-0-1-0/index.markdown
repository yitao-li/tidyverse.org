---
title: themis 0.1.0
author: Emil Hvitfeldt
date: '2020-02-11'
slug: themis-0-1-0
description: 
  themis 0.1.0 is now available on CRAN. Provides additional steps for recipes
  to deal with unbalanced data.
categories:
  - package
tags: [tidymodels, themis]
photo: 
  url: https://unsplash.com/photos/RtDwtRDvYQg
  author: Roman Kraft
---



We're chuffed to announce the release of [themis](https://github.com/tidymodels/themis) on CRAN. [themis](https://tidymodels.github.io/themis/) implements a collection of new steps for the [recipes](https://github.com/tidymodels/recipes) package to deal with unbalanced data. themis is still in early development so any and all feedback is highly appreciated.


```r
library(modeldata)
library(recipes)
library(themis)
```

In a classification context, a dataset is said to be unbalanced if there is an unequal number of observations in each class. Many models perform best when the number of observations is equal and, thus, tend to struggle with unbalanced data.

The steps in this package can be divided into two camps:

- Ones that remove observations from the majority class(es), and
- Ones that add observations to the minority class(es).

You can do more than one action, and, thus, are able to mix and match by for example first removing observations from the majority class followed by adding observations to the minority class to achieve the balance you want. 

## Hybrid-sampling steps

Hybrid-sampling involves adding observations to the minority class. This can be done in multiple ways, one way is to sample existing data points like `step_upsample()` does, another way is to synthetically generate new points based on existing points, `step_smote()` and related steps uses k nearest neighbor information to generate new points. Currently `step_upsample()`, `step_smote()`, `step_bsmote()`, `step_adasyn()`, and `step_rose()` are available. All steps have references in their respective help pages. They have slightly different requirements according to the data they can handle; most need all numeric with no missing data, but those requirements can and should be handled by previous steps.

In the following example, let's look at the `okc` dataset. and we can see that the imbalance is 1-to-6.


```r
data("okc")

table(okc$Class)
#> 
#>  stem other 
#>  9539 50316
```

We will use `age`, `diet` and `height` in modeling to predict `Class`. Since `diet` is a factor, we first need to dummify it before we normalize and perform mean imputation to handle all the missing data.


```r
rec <- recipe(Class ~ age + diet + height, data = okc) %>%
  step_unknown(diet) %>%
  step_dummy(diet) %>%
  step_normalize(all_predictors()) %>%
  step_meanimpute(all_predictors()) %>%
  step_smote(Class) 

rec %>%
  prep() %>%
  juice() %>%
  pull(Class) %>%
  table()
#> .
#>  stem other 
#> 50316 50316
```

And we see that the resulting dataset has a perfectly even distribution. All the hybrid-sampling steps share the parameter `over_ratio`, which specifies the desired ratio between the biggest class and the smallest class. It defaults to 1 for an even distribution but can be set to something like `0.5` to have the minority class become half the size of the majority class.


```r
rec <- recipe(Class ~ age + diet + height, data = okc) %>%
  step_unknown(diet) %>%
  step_dummy(diet) %>%
  step_normalize(all_predictors()) %>%
  step_meanimpute(all_predictors()) %>%
  step_smote(Class, over_ratio = 0.5) 

rec %>%
  prep() %>%
  juice() %>%
  pull(Class) %>%
  table()
#> .
#>  stem other 
#> 25158 50316
```

## Under-sampling steps

Under-sampling is removing observations from the majority class. Currently `step_downsample()`, `step_nearmiss()` and `step_tomek()` are available. These steps should have the same user experience as the previous steps as they have a similar shared parameter `under_ratio` which is the ratio between the smallest and the biggest class. Simply using `step_downsample()` randomly removes samples in the majority classes to get them to be the same size as the smallest class.


```r
rec <- recipe(Class ~ age + diet + height, data = okc) %>%
  step_unknown(diet) %>%
  step_dummy(diet) %>%
  step_normalize(all_predictors()) %>%
  step_meanimpute(all_predictors()) %>%
  step_downsample(Class) 

rec %>%
  prep() %>%
  juice() %>%
  pull(Class) %>%
  table()
#> .
#>  stem other 
#>  9539  9539
```
