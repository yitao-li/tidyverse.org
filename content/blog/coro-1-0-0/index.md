---
output: hugodown::hugo_document

slug: coro-1-0-0
title: Coroutines for R!
date: 2020-12-10
author: Lionel Henry
description: >
    A 2-3 sentence description of the post that appears on the articles page.
    This can be omitted if it would just recapitulate the title.

photo:
  url: https://unsplash.com/photos/n6vS3xlnsCc
  author: Kelley Bozarth

categories: [package]
tags: []
rmd_hash: 4cd5680eeb8ed897

---

<!--
TODO:
* [ ] Pick category and tags (see existing with `post_tags()`)
* [ ] Find photo & update yaml metadata
* [ ] Create `thumbnail-sq.jpg`; height and width should be equal
* [ ] Create `thumbnail-wd.jpg`; width should be >5x height
* [ ] `hugodown::use_tidy_thumbnail()`
* [ ] Add intro sentence
* [ ] `use_tidy_thanks()`
-->

It is with sprouting merriment that we announce the first release of [coro](https://coro.r-lib.org/)! coro implements coroutines for R, a kind of functions that can suspend and resume themselves before their final [`return()`](https://rdrr.io/r/base/function.html). Coroutines have proved to be extremely useful in other languages for creating complex lazy sequences (with generators) and concurrency code that is easy for humans to read and write (with async functions).

You can install coro from CRAN with:

<div class="highlight">

<pre class='chroma'><code class='language-r' data-lang='r'><span class='nf'><a href='https://rdrr.io/r/utils/install.packages.html'>install.packages</a></span><span class='o'>(</span><span class='s'>"coro"</span><span class='o'>)</span>
</code></pre>

</div>

This blog post will introduce the two sorts of coroutines implemented in coro, generators and coroutines. It will also demonstrate how to use these coroutines in your workflow for existing packages like reticulate and shiny.

<div class="highlight">

<pre class='chroma'><code class='language-r' data-lang='r'><span class='kr'><a href='https://rdrr.io/r/base/library.html'>library</a></span><span class='o'>(</span><span class='nv'><a href='https://github.com/r-lib/coro'>coro</a></span><span class='o'>)</span>
</code></pre>

</div>

## Coroutines

Coroutines are a special sort of functions that can suspend themselves and resume later on. There are two kinds:

-   Generators which produce values for complex sequences lazily (that is, on demand). These sequences may be infinite or may produce values that are too large to be held in memory all at once.

-   Async functions which work together with a scheduler of concurrent functions. Async functions suspend themselves when they can't make progress until some computation has finished or some event has occurred. The scheduler gets back control and is free to launch a new concurrent computation or resume a suspended async function that is now ready to make progress.

The common property of all coroutines is that they start to perform some work, decide by themselves that they have done enough for now, and return an intermediate value to their caller. It is the caller which decides when to call them again to do some more work. Whereas generators communicate intermediate values to you, the user, async functions exclusively communicate in the background with a scheduler of concurrent computations.

## Generators

The term "generator" may refer to two sorts of functions:

-   A generator factory
-   A generator instance

[`coro::generator()`](https://rdrr.io/pkg/coro/man/generator.html) creates generator factories. These factories in turn create fresh generator instances. Generator factories look like normal functions for the most part, except that you can [`yield()`](https://rdrr.io/pkg/coro/man/yield.html) values.

<div class="highlight">

<pre class='chroma'><code class='language-r' data-lang='r'><span class='c'># Create a generator factory</span>
<span class='nv'>generate_abc</span> <span class='o'>&lt;-</span> <span class='nf'><a href='https://rdrr.io/pkg/coro/man/generator.html'>generator</a></span><span class='o'>(</span><span class='kr'>function</span><span class='o'>(</span><span class='o'>)</span> <span class='o'>&#123;</span>
  <span class='nf'><a href='https://rdrr.io/pkg/coro/man/yield.html'>yield</a></span><span class='o'>(</span><span class='s'>"a"</span><span class='o'>)</span>
  <span class='nf'><a href='https://rdrr.io/pkg/coro/man/yield.html'>yield</a></span><span class='o'>(</span><span class='s'>"b"</span><span class='o'>)</span>
  <span class='s'>"c"</span>
<span class='o'>&#125;</span><span class='o'>)</span>
</code></pre>

</div>

The other difference with normal functions is that generator factories don't return a value immediately. They return a function object, a fresh generator instance.

<div class="highlight">

<pre class='chroma'><code class='language-r' data-lang='r'><span class='c'># Create a generator instance</span>
<span class='nv'>abc</span> <span class='o'>&lt;-</span> <span class='nf'>generate_abc</span><span class='o'>(</span><span class='o'>)</span>

<span class='nv'>abc</span>

<span class='c'>#&gt; &lt;generator/instance&gt;</span>
<span class='c'>#&gt; function() &#123;</span>
<span class='c'>#&gt;   yield("a")</span>
<span class='c'>#&gt;   yield("b")</span>
<span class='c'>#&gt;   "c"</span>
<span class='c'>#&gt; &#125;</span>
</code></pre>

</div>

Calling a generator *yields* a value. It can yield as many time as necessary. The last value is *returned*, after which the generator is stale and returns an exhaustion value.

<div class="highlight">

<pre class='chroma'><code class='language-r' data-lang='r'><span class='nf'>abc</span><span class='o'>(</span><span class='o'>)</span>

<span class='c'>#&gt; [1] "a"</span>


<span class='nf'>abc</span><span class='o'>(</span><span class='o'>)</span>

<span class='c'>#&gt; [1] "b"</span>


<span class='nf'>abc</span><span class='o'>(</span><span class='o'>)</span>

<span class='c'>#&gt; [1] "c"</span>


<span class='nf'>abc</span><span class='o'>(</span><span class='o'>)</span>

<span class='c'>#&gt; exhausted</span>


<span class='nf'>abc</span><span class='o'>(</span><span class='o'>)</span>

<span class='c'>#&gt; exhausted</span>


<span class='nf'><a href='https://rdrr.io/pkg/coro/man/iterator.html'>is_exhausted</a></span><span class='o'>(</span><span class='nf'>abc</span><span class='o'>(</span><span class='o'>)</span><span class='o'>)</span>

<span class='c'>#&gt; [1] TRUE</span>
</code></pre>

</div>

Generators can [`yield()`](https://rdrr.io/pkg/coro/man/yield.html) flexibly inside `if` branches, loops, or [`tryCatch()`](https://rdrr.io/r/base/conditions.html) expressions. For instance we could rewrite the `abc` generator with a loop:

<div class="highlight">

<pre class='chroma'><code class='language-r' data-lang='r'><span class='nv'>generate_abc</span> <span class='o'>&lt;-</span> <span class='nf'><a href='https://rdrr.io/pkg/coro/man/generator.html'>generator</a></span><span class='o'>(</span><span class='kr'>function</span><span class='o'>(</span><span class='o'>)</span> <span class='o'>&#123;</span>
 <span class='kr'>for</span> <span class='o'>(</span><span class='nv'>x</span> <span class='kr'>in</span> <span class='nv'>letters</span><span class='o'>[</span><span class='m'>1</span><span class='o'>:</span><span class='m'>3</span><span class='o'>]</span><span class='o'>)</span> <span class='o'>&#123;</span>
   <span class='nf'><a href='https://rdrr.io/pkg/coro/man/yield.html'>yield</a></span><span class='o'>(</span><span class='nv'>x</span><span class='o'>)</span>
 <span class='o'>&#125;</span>
<span class='o'>&#125;</span><span class='o'>)</span>
</code></pre>

</div>

### Working with iterators

Technically, generator instances are **iterator** functions. Calling them repeatedly advances the iteration step by step until exhaustion. coro provides two helpers that make it easy to work with iterator functions.

-   [`coro::loop()`](https://rdrr.io/pkg/coro/man/collect.html) instruments `for` so that it understands how to loop over these iterators:

    <div class="highlight">

    <pre class='chroma'><code class='language-r' data-lang='r'><span class='nf'><a href='https://rdrr.io/pkg/coro/man/collect.html'>loop</a></span><span class='o'>(</span><span class='kr'>for</span> <span class='o'>(</span><span class='nv'>x</span> <span class='kr'>in</span> <span class='nf'>generate_abc</span><span class='o'>(</span><span class='o'>)</span><span class='o'>)</span> <span class='o'>&#123;</span>
      <span class='nf'><a href='https://rdrr.io/r/base/print.html'>print</a></span><span class='o'>(</span><span class='nf'><a href='https://rdrr.io/r/base/chartr.html'>toupper</a></span><span class='o'>(</span><span class='nv'>x</span><span class='o'>)</span><span class='o'>)</span>
    <span class='o'>&#125;</span><span class='o'>)</span>

    <span class='c'>#&gt; [1] "A"</span>
    <span class='c'>#&gt; [1] "B"</span>
    <span class='c'>#&gt; [1] "C"</span>
    </code></pre>

    </div>

-   [`coro::collect()`](https://rdrr.io/pkg/coro/man/collect.html) loops over the iterator and collects all values in a list:

    <div class="highlight">

    <pre class='chroma'><code class='language-r' data-lang='r'><span class='nf'><a href='https://rdrr.io/pkg/coro/man/collect.html'>collect</a></span><span class='o'>(</span><span class='nf'>generate_abc</span><span class='o'>(</span><span class='o'>)</span><span class='o'>)</span>

    <span class='c'>#&gt; [[1]]</span>
    <span class='c'>#&gt; [1] "a"</span>
    <span class='c'>#&gt; </span>
    <span class='c'>#&gt; [[2]]</span>
    <span class='c'>#&gt; [1] "b"</span>
    <span class='c'>#&gt; </span>
    <span class='c'>#&gt; [[3]]</span>
    <span class='c'>#&gt; [1] "c"</span>
    </code></pre>

    </div>

In a generator function, all `for` loops natively understand iterators. This makes it easy to chain generators. A generator that takes other generators as input to modify their values is called an *adaptor*:

<div class="highlight">

<pre class='chroma'><code class='language-r' data-lang='r'><span class='nv'>adapt_prefix</span> <span class='o'>&lt;-</span> <span class='nf'><a href='https://rdrr.io/pkg/coro/man/generator.html'>generator</a></span><span class='o'>(</span><span class='kr'>function</span><span class='o'>(</span><span class='nv'>it</span>, <span class='nv'>prefix</span><span class='o'>)</span> <span class='o'>&#123;</span>
  <span class='kr'>for</span> <span class='o'>(</span><span class='nv'>x</span> <span class='kr'>in</span> <span class='nv'>it</span><span class='o'>)</span> <span class='o'>&#123;</span>
    <span class='nf'><a href='https://rdrr.io/pkg/coro/man/yield.html'>yield</a></span><span class='o'>(</span><span class='nf'><a href='https://rdrr.io/r/base/paste.html'>paste0</a></span><span class='o'>(</span><span class='nv'>prefix</span>, <span class='nv'>x</span><span class='o'>)</span><span class='o'>)</span>
  <span class='o'>&#125;</span>
<span class='o'>&#125;</span><span class='o'>)</span>

<span class='kr'><a href='https://rdrr.io/r/base/library.html'>library</a></span><span class='o'>(</span><span class='nv'><a href='https://magrittr.tidyverse.org'>magrittr</a></span><span class='o'>)</span>

<span class='nf'>generate_abc</span><span class='o'>(</span><span class='o'>)</span> <span class='o'>%&gt;%</span> <span class='nf'>adapt_prefix</span><span class='o'>(</span><span class='s'>"foo_"</span><span class='o'>)</span> <span class='o'>%&gt;%</span> <span class='nf'><a href='https://rdrr.io/pkg/coro/man/collect.html'>collect</a></span><span class='o'>(</span><span class='o'>)</span>

<span class='c'>#&gt; [[1]]</span>
<span class='c'>#&gt; [1] "foo_a"</span>
<span class='c'>#&gt; </span>
<span class='c'>#&gt; [[2]]</span>
<span class='c'>#&gt; [1] "foo_b"</span>
<span class='c'>#&gt; </span>
<span class='c'>#&gt; [[3]]</span>
<span class='c'>#&gt; [1] "foo_c"</span>
</code></pre>

</div>

### Compatibility with reticulate

Python iterators from the [reticulate](https://rstudio.github.io/reticulate/) package are fully compatible with coro. Let's create a Python generator for the first `n` integers:

<div class="highlight">

<pre class='chroma'><code class='language-r' data-lang='r'><span class='nf'><a href='https://rdrr.io/r/base/message.html'>suppressMessages</a></span><span class='o'>(</span>
  <span class='kr'><a href='https://rdrr.io/r/base/library.html'>library</a></span><span class='o'>(</span><span class='nv'><a href='https://github.com/rstudio/reticulate'>reticulate</a></span><span class='o'>)</span>
<span class='o'>)</span>

<span class='nf'><a href='https://rdrr.io/pkg/reticulate/man/py_run.html'>py_run_string</a></span><span class='o'>(</span><span class='s'>"
def first_n(n):
    num = 1
    while num &lt;= n:
        yield num
        num += 1
"</span><span class='o'>)</span>
</code></pre>

</div>

You can [`loop()`](https://rdrr.io/pkg/coro/man/collect.html) over iterators created by this generator:

<div class="highlight">

<pre class='chroma'><code class='language-r' data-lang='r'><span class='nv'>first_3</span> <span class='o'>&lt;-</span> <span class='nv'>py</span><span class='o'>$</span><span class='nf'>first_n</span><span class='o'>(</span><span class='m'>3</span><span class='o'>)</span>

<span class='nf'><a href='https://rdrr.io/pkg/coro/man/collect.html'>loop</a></span><span class='o'>(</span><span class='kr'>for</span> <span class='o'>(</span><span class='nv'>x</span> <span class='kr'>in</span> <span class='nv'>first_3</span><span class='o'>)</span> <span class='o'>&#123;</span>
  <span class='nf'><a href='https://rdrr.io/r/base/print.html'>print</a></span><span class='o'>(</span><span class='nv'>x</span> <span class='o'>*</span> <span class='m'>2</span><span class='o'>)</span>
<span class='o'>&#125;</span><span class='o'>)</span>

<span class='c'>#&gt; [1] 2</span>
<span class='c'>#&gt; [1] 4</span>
<span class='c'>#&gt; [1] 6</span>
</code></pre>

</div>

You can [`collect()`](https://rdrr.io/pkg/coro/man/collect.html) the values:

<div class="highlight">

<pre class='chroma'><code class='language-r' data-lang='r'><span class='nf'><a href='https://rdrr.io/pkg/coro/man/collect.html'>collect</a></span><span class='o'>(</span><span class='nv'>py</span><span class='o'>$</span><span class='nf'>first_n</span><span class='o'>(</span><span class='m'>3</span><span class='o'>)</span><span class='o'>)</span>

<span class='c'>#&gt; [[1]]</span>
<span class='c'>#&gt; [1] 1</span>
<span class='c'>#&gt; </span>
<span class='c'>#&gt; [[2]]</span>
<span class='c'>#&gt; [1] 2</span>
<span class='c'>#&gt; </span>
<span class='c'>#&gt; [[3]]</span>
<span class='c'>#&gt; [1] 3</span>
</code></pre>

</div>

And you can chain them with coro generators:

<div class="highlight">

<pre class='chroma'><code class='language-r' data-lang='r'><span class='nv'>adapt_plus</span> <span class='o'>&lt;-</span> <span class='nf'><a href='https://rdrr.io/pkg/coro/man/generator.html'>generator</a></span><span class='o'>(</span><span class='kr'>function</span><span class='o'>(</span><span class='nv'>it</span>, <span class='nv'>n</span><span class='o'>)</span> <span class='o'>&#123;</span>
  <span class='kr'>for</span> <span class='o'>(</span><span class='nv'>x</span> <span class='kr'>in</span> <span class='nv'>it</span><span class='o'>)</span> <span class='nf'><a href='https://rdrr.io/pkg/coro/man/yield.html'>yield</a></span><span class='o'>(</span><span class='nv'>x</span> <span class='o'>+</span> <span class='nv'>n</span><span class='o'>)</span>
<span class='o'>&#125;</span><span class='o'>)</span>

<span class='nv'>py</span><span class='o'>$</span><span class='nf'>first_n</span><span class='o'>(</span><span class='m'>3</span><span class='o'>)</span> <span class='o'>%&gt;%</span> <span class='nf'>adapt_plus</span><span class='o'>(</span><span class='m'>10</span><span class='o'>)</span> <span class='o'>%&gt;%</span> <span class='nf'><a href='https://rdrr.io/pkg/coro/man/collect.html'>collect</a></span><span class='o'>(</span><span class='o'>)</span>

<span class='c'>#&gt; [[1]]</span>
<span class='c'>#&gt; [1] 11</span>
<span class='c'>#&gt; </span>
<span class='c'>#&gt; [[2]]</span>
<span class='c'>#&gt; [1] 12</span>
<span class='c'>#&gt; </span>
<span class='c'>#&gt; [[3]]</span>
<span class='c'>#&gt; [1] 13</span>
</code></pre>

</div>

### When should I use generators?

Generators are important in Python because they provide a flexible way of creating iterators and these are at the heart of the language. However, whereas Python is scalar oriented, R is a vector oriented language. Also, R is a functional language which makes iterators a bit at odds because they are *stateful*. Advancing an iterator changes the world irremediably. If you want to produce the last value again, you need to start over. For these reasons, generators are likely not the most appropriate way of solving your problems. In most cases it will be more efficient and natural to work with vectorised or functional idioms.

On the other hand, vectorised idioms do not work well when:

-   The data doesn't fit in memory. Infinite sequences are an extreme case of this. When you can't work with all the data at once, it must be chunked into more manageable slices.

-   The sequence is complex or you don't need to compute all of it in advance.

Generators are a good way of structuring computations on chunked data and lazy sequences.

## Async functions

The most useful application of generators is to create concurrent computations that yield to each other so that they can both make progress in a given lapse of time.

You won't see intermediate values produced by an async function. The suspension mechanism is designed to be transparent to the user and give the illusion of sequential evaluation.

## Acknowledgements

