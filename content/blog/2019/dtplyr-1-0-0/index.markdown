---
title: dtplyr 1.0.0
author: Hadley Wickham
date: '2019-11-12'
slug: dtplyr-1-0-0
description: |
  A total rewrite of dtplyr is now available on CRAN; it performs
  computation lazily (like dbplyr), making it much more performant.
categories:
  - package
tags:
  - dtplyr
  - dplyr
  - tidyverse
photo:
  url: https://unsplash.com/photos/h13Y8vyIXNU
  author: Jay Ruzesky
---



I'm very excited to announce that [dtplyr](https://dtplyr.tidyverse.org) 1.0.0 is now on CRAN. dtplyr provides a [data.table](http://r-datatable.com/) backend for dplyr, allowing you to write dplyr code that is automatically translated to the equivalent data.table code.  

dtplyr 1.0.0 gives you the speed of data.table with the syntax of dplyr, unlocking the value of data.table to every user of dplyr. Of course, if you know data.table, you can still write it directly, just as we expect SQL experts to continue to write SQL rather than having [dbplyr](http://dbplyr.tidyverse.org/) generate it for them. Understanding these foundational tools is particularly important if you want to eke out every last drop of performance.

This version of dtplyr is a complete rewrite which allows it to generate significantly more performant translations. dtplyr now works like [dbplyr](https://dbplyr.tidyverse.org), where computation only happens when requested by  `as.data.table()`, `as.data.frame()` or `as_tibble()` (this idea can also be seen in the [table.express](https://github.com/asardaes/table.express) and [rqdatatable](https://github.com/WinVector/rqdatatable/) packages). Unfortunately, this rewrite breaks pretty much all existing uses of dtplyr. But frankly, the prior version of dtplyr was not very good and few people used it, so a major overhaul should break little code in the wild.

In this article, I'll introduce you to the basic usage of dtplyr, talk about some of the performance implications, and show off some of the translations that I'm most proud of.

## Usage

To use dtplyr, you must at least load dtplyr and dplyr. You might also want to load [data.table](http://r-datatable.com/) so you can access the other goodies that it provides:


```r
library(data.table)
library(dtplyr)
library(dplyr, warn.conflicts = FALSE)
```

Then use `lazy_dt()` to create a "lazy" data table that tracks the operations performed on it:


```r
mtcars2 <- lazy_dt(mtcars)
```

You can use dplyr verbs with this object as if it was a data frame. But there's a big difference behind the scenes: instead of immediately performing the operation, dtplyr just records what you're doing so when needed it can generate a single, efficient, data.table statement. You can preview the transformation (including the generated data.table code) by printing the result:


```r
mtcars2 %>% 
  filter(wt < 5) %>% 
  mutate(l100k = 235.21 / mpg) %>% # liters / 100 km
  group_by(cyl) %>% 
  summarise(l100k = mean(l100k))
#> Source: local data table [?? x 2]
#> Call:   `_DT1`[wt < 5][, `:=`(l100k = 235.21/mpg)][, .(l100k = mean(l100k)), 
#>     keyby = .(cyl)]
#> 
#>     cyl l100k
#>   <dbl> <dbl>
#> 1     4  9.05
#> 2     6 12.0 
#> 3     8 14.9 
#> 
#> # Use as.data.table()/as.data.frame()/as_tibble() to access results
```

Generally, however, you should reserve this preview for exploration and debugging, and instead use `as.data.table()`, `as.data.frame()`, or `as_tibble()` to indicate that you're done writing the transformation and want to access the results:


```r
mtcars2 %>% 
  filter(wt < 5) %>% 
  mutate(l100k = 235.21 / mpg) %>% # liters / 100 km
  group_by(cyl) %>% 
  summarise(l100k = mean(l100k)) %>% 
  as_tibble()
#> # A tibble: 3 x 2
#>     cyl l100k
#>   <dbl> <dbl>
#> 1     4  9.05
#> 2     6 12.0 
#> 3     8 14.9
```

## Performance

How fast is dtplyr? data.table is generally faster than dplyr, but dtplyr has to do some work to perform the translation, so it's reasonable to ask if it's worth it. Do the benefits of using data.table outweigh the cost of the automated translation? My experimentation suggests that it is: the cost of translation is low, and independent of the size of the data. In this section, I'll explore the performance trade-off through three lenses: translation cost, copies, and interface mismatch.


### Translation cost

Each dplyr verb must do some work to convert dplyr syntax to data.table syntax. We can use the [bench](http://bench.r-lib.org/) package to time the cost of the four-step pipeline that I used above:
  

```r
bench::mark(
  translate = mtcars2 %>% 
    filter(wt < 5) %>% 
    mutate(l100k = 235.21 / mpg) %>% # liters / 100 km
    group_by(cyl) %>% 
    summarise(l100k = mean(l100k))
)
#> # A tibble: 1 x 6
#>   expression      min   median `itr/sec` mem_alloc `gc/sec`
#>   <bch:expr> <bch:tm> <bch:tm>     <dbl> <bch:byt>    <dbl>
#> 1 translate     787µs    969µs     1028.      280B     26.4
```

Because this pipeline does not use `as.data.table()` or `print()` it only generates the data.table code, it doesn't run it, so we're timing the translation cost. The translation cost scales with the complexity of the pipeline, not the size of the data, so these timings will apply regardless of the size of the underlying data.

My intial experiments suggest that the translation cost is typically a few milliseconds. Since the computational cost increases with the size of the data, the translation cost becomes a smaller proportion of the total as the data size grows, suggesting the dtplyr does not impose a significant overhead on top of data.table.

Take the following example, which uses the large nycflights13 dataset. This isn't really big enough for data.table to really shine, but it's about as big as you can get in an R package. Here I'm going to compute the average arrival delay by destination. It takes raw dplyr about 40ms to do the work. Again, the dtplyr translation is fast, around 1ms, and then computation using data.table only takes about 20ms, almost twice as fast as dplyr.


```r
library(nycflights13)
flights_dt <- lazy_dt(flights)

delay_by_dest <- function(df) {
  df %>%
    filter(!is.na(arr_delay)) %>% 
    group_by(dest) %>% 
    summarise(n = n(), delay = mean(arr_delay))
}

bench::mark(
  flights %>% delay_by_dest(),
  flights_dt %>% delay_by_dest(),
  flights_dt %>% delay_by_dest() %>% as_tibble(),
  check = FALSE
)
#> # A tibble: 3 x 6
#>   expression                                         min  median `itr/sec`
#>   <bch:expr>                                     <bch:t> <bch:t>     <dbl>
#> 1 flights %>% delay_by_dest()                     35.7ms    36ms      27.8
#> 2 flights_dt %>% delay_by_dest()                 671.9µs 824.6µs    1230. 
#> 3 flights_dt %>% delay_by_dest() %>% as_tibble()  18.7ms  20.2ms      48.0
#> # … with 2 more variables: mem_alloc <bch:byt>, `gc/sec` <dbl>
```

### Copies

There is one place where dtplyr does have to add overhead: when translations would generate data.table code that modifies the input in place, like `mutate()`. dtplyr matches dplyr semantics (which never modifies in place), so most expressions involving `mutate()` must make a copy:
  

```r
mtcars2 %>% 
  mutate(x = x * 2) %>% 
  show_query()
#> copy(`_DT1`)[, `:=`(x = x * 2)]
```

However, dtplyr never generates more than one copy (no matter how many mutates you use), and it also recognises many situations where data.table creates an implicit copy:


```r
mtcars2 %>% 
  mutate(y = x * 2) %>% 
  mutate(z = y * 2) %>% 
  show_query()
#> copy(`_DT1`)[, `:=`(y = x * 2)][, `:=`(z = y * 2)]

mtcars2 %>% 
  filter(x == 1) %>% 
  mutate(x = x * 2) %>% 
  show_query()
#> `_DT1`[x == 1][, `:=`(x = x * 2)]
```

However, if you have very datasets, creating a deep copy can be expensive. dtplyr allows you to opt out by setting `immutable = FALSE`. This ensures that dtplyr never makes a copy:


```r
mtcars3 <- lazy_dt(as.data.table(mtcars), immutable = FALSE)

mtcars3 %>% 
  mutate(x2 = x * 2) %>% 
  show_query()
#> `_DT3`[, `:=`(x2 = x * 2)]
```

### Interface mismatch

The hardest overhead to measure is the cost of interface mismatch, i.e. where data.table has features that dplyr doesn't. For example, there's no way to express cross- or rolling-joins with dplyr, so there's no way to generate efficient data.table code for these cases. It's hard to estimate this cost, but it's something that we think about when considering what features to add to dplyr next.

## Translation

If you're familiar with data.table, you might be interested to learn more about how the translation works. Here I'll show a few things I think are particularly interesting, using `show_query()`. 


```r
df <- data.frame(a = 1:5, b = 1:5, c = 1:5, d = 1:5)
dt <- lazy_dt(df)
```

Most uses of the basic dplyr verbs generate calls to `[.data.table`:


```r
dt %>% select(-c, -d) %>% show_query()
#> `_DT4`[, .(a, b)]
dt %>% summarise(x = mean(x)) %>% show_query()
#> `_DT4`[, .(x = mean(x))]
dt %>% mutate(x = a + b) %>% show_query()
#> copy(`_DT4`)[, `:=`(x = a + b)]
dt %>% filter(a == 1) %>% show_query()
#> `_DT4`[a == 1]
dt %>% arrange(a, desc(b)) %>% show_query()
#> `_DT4`[order(a, desc(b))]
```

As do simple left and right joins:


```r
dt2 <- lazy_dt(data.frame(a = 1, y = 1, z = 1))
dt %>% left_join(dt2, by = "a") %>% show_query()
#> `_DT5`[`_DT4`, on = .(a), allow.cartesian = TRUE]
dt %>% right_join(dt2, by = "a") %>% show_query()
#> `_DT4`[`_DT5`, on = .(a), allow.cartesian = TRUE]
```

Where possible, dtplyr will collapse multiple calls to `[`:


```r
dt %>% 
  filter(a == 1) %>% 
  select(-a) %>% 
  show_query()
#> `_DT4`[a == 1, .(b, c, d)]

dt %>% 
  left_join(dt2, by = "a") %>% 
  select(a, b, z) %>% 
  show_query()
#> `_DT5`[`_DT4`, .(a, b, z), on = .(a), allow.cartesian = TRUE]
```

But note that the order is important, as a `select()` followed by a `filter()` has to generate two statements:


```r
dt %>% 
  select(a = b) %>% 
  filter(a == 1) %>% 
  show_query()
#> `_DT4`[, .(a = b)][a == 1]
```


When you mix basic dplyr verbs with `group_by()`, dtplyr adds the `keyby` argument:


```r
dt %>% 
  group_by(a) %>% 
  summarise(b = mean(b)) %>% 
  show_query()
#> `_DT4`[, .(b = mean(b)), keyby = .(a)]
```

And when filtering, this automatically uses `.SD`:


```r
dt %>% 
  group_by(a) %>% 
  filter(b < mean(b)) %>% 
  show_query()
#> `_DT4`[, .SD[b < mean(b)], keyby = .(a)]
```

You can learn more in [`vignette("translation")`](https://dtplyr.tidyverse.org/articles/translation.html). 

There are a couple of limitations that I hope to address in the next version of dtplyr. Currently, you can't translate [the `_if` variants](https://github.com/tidyverse/dtplyr/issues/109), and there is weak support for the [`group_` functions](https://github.com/tidyverse/dtplyr/issues/108). If you discover other functions that don't work as you expect, [please file an issue!](https://github.com/tidyverse/dtplyr/issues/new/choose).

## Acknowledgements

Big thanks to the data.table community, particularly [Michael Chirico](https://github.com/MichaelChirico), for their help educating me on the best way to translate dplyr code into performant, idiomatic, data.table code.

I'd also like to thank everyone to helped make this release happen through their contributions on GitHub: [&#x0040;AlanFeder](https://github.com/AlanFeder), [&#x0040;batpigandme](https://github.com/batpigandme), [&#x0040;benjaminleroy](https://github.com/benjaminleroy), [&#x0040;clayphan](https://github.com/clayphan), [&#x0040;ColinFay](https://github.com/ColinFay), [&#x0040;daranzolin](https://github.com/daranzolin), [&#x0040;edgararuiz](https://github.com/edgararuiz), [&#x0040;franknarf1](https://github.com/franknarf1), [&#x0040;hadley](https://github.com/hadley), [&#x0040;hlynurhallgrims](https://github.com/hlynurhallgrims), [&#x0040;hope-data-science](https://github.com/hope-data-science), [&#x0040;ianmcook](https://github.com/ianmcook), [&#x0040;jl5000](https://github.com/jl5000), [&#x0040;jonthegeek](https://github.com/jonthegeek), [&#x0040;JoshuaSturm](https://github.com/JoshuaSturm), [&#x0040;lenay12](https://github.com/lenay12), [&#x0040;MichaelChirico](https://github.com/MichaelChirico), [&#x0040;nlbjan1](https://github.com/nlbjan1), [&#x0040;quantitative-technologies](https://github.com/quantitative-technologies), [&#x0040;richpauloo](https://github.com/richpauloo), [&#x0040;S-UP](https://github.com/S-UP), [&#x0040;tmastny](https://github.com/tmastny), [&#x0040;TobiRoby](https://github.com/TobiRoby), [&#x0040;tomazweiss](https://github.com/tomazweiss), [&#x0040;torema-ed](https://github.com/torema-ed), [&#x0040;Vidaringa](https://github.com/Vidaringa), [&#x0040;vlahm](https://github.com/vlahm), [&#x0040;vspinu](https://github.com/vspinu), [&#x0040;xiaodaigh](https://github.com/xiaodaigh), and [&#x0040;yiqinfu](https://github.com/yiqinfu).
