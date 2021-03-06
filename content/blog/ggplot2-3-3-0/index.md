---
title: ggplot2 3.3.0
author: Thomas Lin Pedersen
date: '2020-03-05'
description: "The next iteration of ggplot2 has just been released on CRAN, packed
  with new \nfeatures and bug fixes. Read all about what's new here.\n"
output: hugodown::hugo_document
photo:
  author: Steve Harvey
  url: https://unsplash.com/photos/xWiXi6wRLGo
slug: ggplot2-3-3-0
categories:
- package
rmd_hash: 4c775a26cdaa575b

---


We're so happy to announce the release of ggplot2 3.3.0 on CRAN. [ggplot2](https://ggplot2.tidyverse.org/) is a system for declaratively creating graphics, based on *The Grammar of Graphics*. You provide the data, tell ggplot2 how to map variables to aesthetics, what graphical primitives to use, and it takes care of the details. The new version can be installed from CRAN using `install.packages("ggplot2")`

This release is substantial, with both big new features and internal rewrites. It contains most of the work [Dewey Dunnington](https://github.com/paleolimbot) did as part of his internship at RStudio, along with contributions from many others. Dewey will continue working on ggplot2 as part of the core team, and we are very lucky to have him. He has written a [post about his internship](https://education.rstudio.com/blog/2019/10/a-summer-of-rstudio-and-ggplot2/) which you should check out to get an overview of all that he has done. 

The [NEWS](https://ggplot2.tidyverse.org/news/index.html) file contains an overview of all the changes in this version—we will only discuss the biggest features below.

## New features
The last ggplot2 release was primarily performance-oriented. This one, on the other hand is packed with features, big and small. There are also a slew of bug-fixes that we won't go into further detail with. Further, we have now removed reshape2 from the dependencies, which removes the indirect dependencies on plyr, stringi, and stringr. We have been shaving off dependencies in ggplot2 over a range of releases, and with the removal of reshape2, we are now close to being as lean as we believe is possible. To read more about our thoughts on dependencies see the [*It depends*](https://www.tidyverse.org/blog/2019/05/itdepends/) blog post.

### Rewrite of axis code
While Dewey has worked on a lot of different parts of the ggplot2 code base, the lion's share has been concerned with a rewrite of the positional-guide (axis) internals. While, at a high level, axes and legends are equivalent (they are both guides used for scales), this has not been true for the underlying code. With this release, we have streamlined the implementation considerably, and paved the way for a full guide rewrite in a future release (guides are one of the last part waiting to be updated to ggproto). Apart from making our life as ggplot2 developers easier, the rewrite also comes with a slew of user-facing improvements and features. All of this is contained in the new `guide_axis()` function that works equivalently to e.g. `guide_legend()`.


```r
library(ggplot2)

p <- ggplot(mpg) +
  geom_bar(aes(x = manufacturer)) + 
  theme(axis.text.x = element_text(size = 11))

# Overlapping labels
plot(p)
```

<img src="figs/unnamed-chunk-1-1.png" width="700px" style="display: block; margin: auto;" />


```r
# Use guide_axis to dodge the labels
p + 
  scale_x_discrete(guide = guide_axis(n.dodge = 2))
```

<img src="figs/unnamed-chunk-2-1.png" width="700px" style="display: block; margin: auto;" />


```r
# Or to remove overlapping labels
p + 
  scale_x_discrete(guide = guide_axis(check.overlap = TRUE))
```

<img src="figs/unnamed-chunk-3-1.png" width="700px" style="display: block; margin: auto;" />

The resolving of overlapping tick labels is designed so that the first and last labels are always shown. It is obviously best suited for continuous or ordered data where the identity of the removed labels can be deduced from the remaining ones.

### New bin scale
While ggplot2 has a lot of different scales, it has only ever had two different scale *types*: continuous and discrete. This all changes with this release, as a new binning scale type has been added. A binning scale takes continuous data and makes it discrete by assigning each data point to a bin. It thus exists somewhere between the two existing scale types and e.g. allows continuous data to be used with discrete palettes. Being a fundamental scale type, it is available to all aesthetics, both positional and otherwise:


```r
p <- ggplot(mpg) + 
  geom_point(aes(displ, cty, size = hwy, colour = hwy))

p + 
  scale_size_binned()
```

<img src="figs/unnamed-chunk-4-1.png" width="700px" style="display: block; margin: auto;" />


```r
p + 
  scale_colour_binned()
```

<img src="figs/unnamed-chunk-5-1.png" width="700px" style="display: block; margin: auto;" />

As can be seen, the legends reflect the binned nature of the scale by placing tick marks at the border between bins. By default the outermost ticks are not shown, indicating that the binning is open in both ends. This behavior can be controlled in the guide.


```r
p + 
  scale_size_binned(guide = guide_bins(show.limits = TRUE))
```

<img src="figs/unnamed-chunk-6-1.png" width="700px" style="display: block; margin: auto;" />

When used with a positional scale it acts in much the same way, placing data at the center of the bin they belong to:


```r
p + 
  scale_x_binned()
```

<img src="figs/unnamed-chunk-7-1.png" width="700px" style="display: block; margin: auto;" />

One of the benefits of this is that it is now a breeze to create histograms with tick-marks between the bars, as a histogram is effectively a binned scale with a bar geom:


```r
ggplot(mpg) + 
  geom_bar(aes(displ)) + 
  scale_x_binned()
```

<img src="figs/unnamed-chunk-8-1.png" width="700px" style="display: block; margin: auto;" />

### Bi-directional geoms and stats
While we are messing with the foundation of ggplot2, we might as well challenge another paradigm, namely that certain geoms have a direction, and to change the direction you'd have to use `coord_flip()`. The prime example of this is a horizontal bar chart:


```r
ggplot(mpg) + 
  geom_bar(aes(x = manufacturer)) + 
  coord_flip()
```

<img src="figs/unnamed-chunk-9-1.png" width="700px" style="display: block; margin: auto;" />

This approach has served ggplot2 well, and will continue to work in the future, but with this release we update all the directional stats and geoms to work in both directions. The direction of the stat/geom is deduced from the aesthetic mapping, so it should simply behave as expected. The example above can thus be rewritten to:


```r
ggplot(mpg) + 
  geom_bar(aes(y = manufacturer))
```

<img src="figs/unnamed-chunk-10-1.png" width="700px" style="display: block; margin: auto;" />

The direction determination is not only looking at which aesthetics are mapped, but also what they are mapped to, so most layers can be determined without ambiguity. In the presence of ambiguity, it is conservative and will default to the standard orientation. If you want to overwrite the direction, either because it fails to detect it, or because the geom/stat is ambiguous by nature, it can be set directly with the `orientation` argument:


```r
ggplot(mpg, aes(displ, hwy)) + 
  geom_point() + 
  geom_smooth(orientation = "y")
```

<img src="figs/unnamed-chunk-11-1.png" width="700px" style="display: block; margin: auto;" />

### More control over aesthetic evaluation
When mapping data to aesthetics we are usually differentiating between two mappings: one for the stat, and one for the geom. We don't think about this that often, because the stat simply accepts the mappings and carries them over to the geom (sometimes with modifications). Sometimes, though, we want to control the mapping that happens between the output of the stat and the geom. The approach to this has been to use `stat()` (and `..var..` notation before that), to indicate that the aesthetic should be evaluated after the stat has been computed.

With this release we expand on these evaluation controls, as well as making them more explicit. There are now three entry points to evaluation: *start*, *after_stat*, and *after_scale*. *start* is the default (i.e. evaluated in the context of the layer data), *after_stat* is like the old `stat()` (i.e. evaluated in the context of the stat output), and *after_scale* is new and specifies that evaluation should occur after the aesthetic has been scaled. The `after_stat()` and `after_scale()` functions are used to mark aesthetics for their respective delayed evaluation (thus soft-deprecating `stat()`):

You would use `after_stat()` to e.g. pick a different computed metric than the default (here _density_ instead of _count_)


```r
ggplot(mpg, aes(displ)) +
  geom_histogram(aes(y = after_stat(density)))
```

<img src="figs/unnamed-chunk-12-1.png" width="700px" style="display: block; margin: auto;" />

And you can use `after_scale()` to assign fill as a variant of the scaled color:


```r
ggplot(mpg, aes(class, hwy)) +
  geom_boxplot(aes(colour = class, fill = after_scale(alpha(colour, 0.4))))
```

<img src="figs/unnamed-chunk-13-1.png" width="700px" style="display: block; margin: auto;" />

Furthermore, it is now possible to perform multiple mapping for the same aesthetic, using the `stage()` function. This can be used to e.g. set alpha on the stroke of a polygon (the default is to only apply alpha to fill for polygon-type geoms):


```r
ggplot(mpg) + 
  geom_bar(
    aes(
      x = drv, 
      colour = stage(start = drv, after_scale = alpha(colour, 0.5))
    ), 
    fill = NA, size = 4
  )
```

<img src="figs/unnamed-chunk-14-1.png" width="700px" style="display: block; margin: auto;" />

`stage()` is definitely not something you need every day, but it does resolve a range of issues (like the one above), and it is nice to have full transparency and control over the aesthetic evaluation.

### More extensible theming
This new feature is particularly relevant for extension developers. The possibility to extend coordinate systems and faceting has created a need to allow new theme elements to be specified (along with inheritance). One could envision e.g. a faceting setup that related panels with arrows, and in order to control the look of the arrows we would need to allow a call like `theme(panel.arrow = element_line(...))`. With this release, we finally provide a way for extension developers to register new theme elements using `register_theme_elements()`. We urge extension developers to namespace the theme elements they create in order to avoid name clashes. If the above facet was part of the ggrelfacet package the registration would look something like this:


```r
register_theme_elements(
  ggrelfacet.panel.arrow = element_line(
    size = 3, arrow = arrow()
  ),
  element_tree = list(
    ggrelfacet.panel.arrow = el_def("element_line", "line")
  )
)
```

The first argument defines the default look of the element, and the second one provides the type and inheritance (must be an `"element_line"` object, and will inherit from the `"line"` element). If possible, you should try to inherit from some base element and only do general modifications, as that will mean that your new theme elements will fit into whatever theme your user is using. Being overly specific will result in elements that may fit into one theme but look out of place in all others

The call above would be added to the package `.onLoad()` function to make sure the theme element was registered when the package was used, but you will of course also need to create extensions that actually use the theme element for it to have any effect.

It is difficult to showcase the functionality of the theme registration without resolving to develop a full new `Coord` or `Facet`, so we won't show it here. The documentation for `register_theme_elements()` have a minimal example, so consult this in order to get guidance into using it correctly for your extension package.

### Better contour calculations
A recurring request has been to allow for filled contours. This has been problematic in the past because `stat_density_2d()` only calculated the outer contour and didn't cut out any inner holes or next level contours. This meant that if either the fill was partly transparent, or if the level contained any holes, the output would not be a true representation of the data. For an example of this, we can look at our favorite volcano:


```r
volcano_long <- data.frame(
  x = as.vector(col(volcano)),
  y = as.vector(row(volcano)),
  z = as.vector(volcano)
)

ggplot(volcano_long, aes(x, y, z = z)) + 
  geom_polygon(aes(fill = stat(level)), alpha = 0.5, stat = "contour") + 
  guides(fill = "legend")
```

<img src="figs/unnamed-chunk-16-1.png" width="700px" style="display: block; margin: auto;" />

There is so much wrong going on here that we won't go through it. In any case, this is all in the past because ggplot2 has moved on to using the new isoband package and now provides a geom for filled contours specifically:


```r
ggplot(volcano_long, aes(x, y, z = z)) + 
  geom_contour_filled(aes(fill = stat(level)), alpha = 0.5)
```

<img src="figs/unnamed-chunk-17-1.png" width="700px" style="display: block; margin: auto;" />

We see that all the issues above have been fixed. Local minima are now represented, and the alpha level is now a true representation of the scale since levels don't overlap and accumulate opacity. Further, each level is correctly denoted as a range.

One thing you might wonder about is the choice of legend. The filled contours are basically showing binned data, and we have just introduced a new binning scale - can this be merged? Yes and no. The binning has to happen at the stat level, since the contour calculation require un-binned data. So binning at the scale level is not possible. But the new bin legends have been written to understand the values created by the filled contour stat (as well as those returned by `cut()`) and can thus be used to show properly formatted discrete scales:


```r
ggplot(volcano_long, aes(x, y, z = z)) + 
  geom_contour_filled(aes(fill = stat(level))) + 
  guides(fill = guide_colorsteps(barheight = unit(10, "cm")))
```

<img src="figs/unnamed-chunk-18-1.png" width="700px" style="display: block; margin: auto;" />

This also mean that it is possible to use pre-binned data with the new bin guide, as long as you adhere to the output format of `cut()`.

### Grab bag
There are also a range of smaller features in this release that may not be earth shattering, but could mean the world to some. First among these is the new ability to position the plot title, subtitle and caption, flush with the left or right side of the full plot, instead of aligned with the plotting area. This is done with the new `plot.title.position` and `plot.caption.position` theme setting, which takes either `"panel"` (default, old-style) or `"plot"` as values:


```r
ggplot(mpg) + 
  geom_point(aes(hwy, displ)) + 
  ggtitle("The placement of this title may surprise you") + 
  theme(plot.title.position = "plot")
```

<img src="figs/unnamed-chunk-19-1.png" width="700px" style="display: block; margin: auto;" />

When to use what is partly a matter of personal taste, but will usually also depend on the context the plot is used in. As a stand-alone figure, the old style will often look best, while the new style has merits when e.g. the figure appears within a longer text. In the end the decision is up to you.

We have been gradually moving to accepting rlang-style anonymous functions and with the last release we thought the process was complete. We missed `stat_summary()` though, and this release rectifies that. Thus, you can now use any type of notation accepted by `rlang::as_function()` (e.g. `~ median(x)`), when passing different summary functions to `stat_summary()`.

Lastly, we have now made a change in how `geom_area()`/`geom_density()` and `geom_ribbon()` are being drawn. Historically they have been simple polygons, which meant that if you wanted to draw the outline you'd get a stroke around it all. This looked weird, and people would generally expect `geom_ribbon()` to only stroke the upper and lower bounds, and `geom_area()`/`geom_density()` to only stroke the upper bound. With this release we move on to a slightly more sophisticated rendering that allows for exactly that. The behavior can be controlled with the `outline.type` argument that can be set to `"both"` for showing upper and lower bounds, `"upper"`/`"lower"` for showing either, or `"full"` to regain the old behavior, should you want that. The new behavior looks like this:


```r
huron <- data.frame(year = 1875:1972, level = as.vector(LakeHuron))
ggplot(huron, aes(year)) + 
  geom_ribbon(aes(ymin = level - 10, ymax = level + 10), fill = "grey", colour = "black")
```

<img src="figs/unnamed-chunk-20-1.png" width="700px" style="display: block; margin: auto;" />


```r
ggplot(diamonds, aes(carat)) +
  geom_density(fill = "grey")
```

<img src="figs/unnamed-chunk-21-1.png" width="700px" style="display: block; margin: auto;" />

## Acknowledgements
Thank you to the 102 people who who contributed issues, code and comments to this release: [&#x0040;abiyug](https://github.com/abiyug), [&#x0040;adisarid](https://github.com/adisarid), [&#x0040;adrowe1](https://github.com/adrowe1), [&#x0040;AmeliaMN](https://github.com/AmeliaMN), [&#x0040;atusy](https://github.com/atusy), [&#x0040;Ax3man](https://github.com/Ax3man), [&#x0040;batpigandme](https://github.com/batpigandme), [&#x0040;brwheeler](https://github.com/brwheeler), [&#x0040;capebulbs](https://github.com/capebulbs), [&#x0040;carywreams](https://github.com/carywreams), [&#x0040;clauswilke](https://github.com/clauswilke), [&#x0040;cneyens](https://github.com/cneyens), [&#x0040;CorradoLanera](https://github.com/CorradoLanera), [&#x0040;cpsievert](https://github.com/cpsievert), [&#x0040;damianooldoni](https://github.com/damianooldoni), [&#x0040;daniel-barnett](https://github.com/daniel-barnett), [&#x0040;davebraze](https://github.com/davebraze), [&#x0040;dkahle](https://github.com/dkahle), [&#x0040;dmaupin12](https://github.com/dmaupin12), [&#x0040;dracodoc](https://github.com/dracodoc), [&#x0040;eliocamp](https://github.com/eliocamp), [&#x0040;EvaMaeRey](https://github.com/EvaMaeRey), [&#x0040;FantacticMisterFox](https://github.com/FantacticMisterFox), [&#x0040;GeospatialDaryl](https://github.com/GeospatialDaryl), [&#x0040;ggrothendieck](https://github.com/ggrothendieck), [&#x0040;hadley](https://github.com/hadley), [&#x0040;Hillerst](https://github.com/Hillerst), [&#x0040;hvaret](https://github.com/hvaret), [&#x0040;iago-pssjd](https://github.com/iago-pssjd), [&#x0040;idno0001](https://github.com/idno0001), [&#x0040;Ilia-Kosenkov](https://github.com/Ilia-Kosenkov), [&#x0040;isteves](https://github.com/isteves), [&#x0040;James-G-Hill](https://github.com/James-G-Hill), [&#x0040;japhir](https://github.com/japhir), [&#x0040;jarauh](https://github.com/jarauh), [&#x0040;jhuntergit](https://github.com/jhuntergit), [&#x0040;jkbest2](https://github.com/jkbest2), [&#x0040;joethorley](https://github.com/joethorley), [&#x0040;jokorn](https://github.com/jokorn), [&#x0040;jtr13](https://github.com/jtr13), [&#x0040;jzadra](https://github.com/jzadra), [&#x0040;kadyb](https://github.com/kadyb), [&#x0040;karawoo](https://github.com/karawoo), [&#x0040;karsfri](https://github.com/karsfri), [&#x0040;katrinleinweber](https://github.com/katrinleinweber), [&#x0040;Kodiologist](https://github.com/Kodiologist), [&#x0040;koenvandenberge](https://github.com/koenvandenberge), [&#x0040;kuriwaki](https://github.com/kuriwaki), [&#x0040;lizlaw](https://github.com/lizlaw), [&#x0040;luispfonseca](https://github.com/luispfonseca), [&#x0040;lwjohnst86](https://github.com/lwjohnst86), [&#x0040;malcolmbarrett](https://github.com/malcolmbarrett), [&#x0040;MaraAlexeev](https://github.com/MaraAlexeev), [&#x0040;markpayneatwork](https://github.com/markpayneatwork), [&#x0040;MartinEarle](https://github.com/MartinEarle), [&#x0040;Maschette](https://github.com/Maschette), [&#x0040;MaxBareiss](https://github.com/MaxBareiss), [&#x0040;mcsiple](https://github.com/mcsiple), [&#x0040;melissakey](https://github.com/melissakey), [&#x0040;microly](https://github.com/microly), [&#x0040;mine-cetinkaya-rundel](https://github.com/mine-cetinkaya-rundel), [&#x0040;mlamias](https://github.com/mlamias), [&#x0040;mluerig](https://github.com/mluerig), [&#x0040;MohoWu](https://github.com/MohoWu), [&#x0040;mpgerstl](https://github.com/mpgerstl), [&#x0040;msberends](https://github.com/msberends), [&#x0040;N1h1l1sT](https://github.com/N1h1l1sT), [&#x0040;nipnipj](https://github.com/nipnipj), [&#x0040;now2014](https://github.com/now2014), [&#x0040;ovvldc](https://github.com/ovvldc), [&#x0040;paciorek](https://github.com/paciorek), [&#x0040;paleolimbot](https://github.com/paleolimbot), [&#x0040;PatrickRobotham](https://github.com/PatrickRobotham), [&#x0040;pmarchand1](https://github.com/pmarchand1), [&#x0040;privefl](https://github.com/privefl), [&#x0040;Prometheus77](https://github.com/Prometheus77), [&#x0040;qxxxd](https://github.com/qxxxd), [&#x0040;RABxx](https://github.com/RABxx), [&#x0040;rcorty](https://github.com/rcorty), [&#x0040;rfarouni](https://github.com/rfarouni), [&#x0040;rowlesmr](https://github.com/rowlesmr), [&#x0040;shrikantsoni88](https://github.com/shrikantsoni88), [&#x0040;smouksassi](https://github.com/smouksassi), [&#x0040;steenharsted](https://github.com/steenharsted), [&#x0040;StefanBRas](https://github.com/StefanBRas), [&#x0040;steveharoz](https://github.com/steveharoz), [&#x0040;stweb75](https://github.com/stweb75), [&#x0040;teunbrand](https://github.com/teunbrand), [&#x0040;ThomasKnecht](https://github.com/ThomasKnecht), [&#x0040;thomasp85](https://github.com/thomasp85), [&#x0040;TimTeaFan](https://github.com/TimTeaFan), [&#x0040;tjmahr](https://github.com/tjmahr), [&#x0040;tmalsburg](https://github.com/tmalsburg), [&#x0040;traversc](https://github.com/traversc), [&#x0040;tungmilan](https://github.com/tungmilan), [&#x0040;vankesteren](https://github.com/vankesteren), [&#x0040;wch](https://github.com/wch), [&#x0040;weiyangtham](https://github.com/weiyangtham), [&#x0040;wfulp](https://github.com/wfulp), [&#x0040;wilfredom](https://github.com/wilfredom), [&#x0040;woodwards](https://github.com/woodwards), and [&#x0040;yutannihilation](https://github.com/yutannihilation).
