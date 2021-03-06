Performant R Programming: Profiling with profvis
================

Recently, we [wrote
about](https://robotwealth.com/rolling-mean-correlations-in-the-tidyverse/)
calculating mean rolling pairwise correlations between the constituent
stocks of an ETF. The tidyverse tools `dplyr` and `slider` solve this
somewhat painful data wrangling operation about as elegantly and
intuitively as possible.

> *Why on earth did you want to do that?*

We’re building a statistical arbitrage strategy that relies on
ETF-driven trading in the constituents. We wrote about [an early foray
into this trade](https://robotwealth.com/revenge-of-the-stock-pickers/)
- we’re now taking it further. More on this in a future post.

> *But what about the problem of scaling it up?*

When we performed this operation on the constituents of the XLF ETF, our
largest intermediate dataframe consisted of around 3-million rows,
easily within the capabilities of modern laptops.

XLF currently holds 68 constituent stocks. So for any day, we have
\(\frac{68*67}{2} = 2,278\) correlations to estimate (67 because we
don’t want the diagonal of the correlation matrix, take half as we
only need its upper or lower triangle).

We calculated five years of rolling correlations, so we had
\(5*250*2,278 = 2,847,500\) correlations in total.

*Piece of cake.*

The problem gets a lot more interesting if we consider the SPY ETF and
its 500 constituents.

For any day, we’d have \(\frac{500*499}{2} = 124,750\) correlations to
estimate. On five years of data, that’s \(5*250*124,750 = 155,937,500\)
correlations in total.

I tried to do all of that at once in memory on my laptop…and failed.

So our original problem of designing the data wrangling pipline to
achieve our goal has now morphed into a problem of overcoming
performance barriers.

There are a number of strategies that could be employed to solve this.
So over a series of posts, we’ll explore the concept of writing
performant R code via various approaches to solving our problem:

  - Getting more RAM by renting a big virtual machine in the cloud
  - Splitting the data into chunks and doing it sequentially in local
    memory (RAM and hard disk)
  - We could parallelise the above, say using `foreach`, but speed isn’t
    really the issue here, it’s RAM
  - Consider different data structures - `data.table`, `Matrix` - which
    will carry less overhead than a regular `data.frame` or `tibble`.  
  - Horizontal scaling with the `future` package
  - R packages for dealing with memory issues: `ff`, `bigmemory`,
    `MonetDB.R`
  - Doing the calculation in Rcpp, since representing data in C++
    carries less overhead than in R
  - Use Spark, a cluster computing platform for farming the problem out
    to multiple machines
  - Use Dataflow, a serverless batch data processing tool available on
    commercial cloud providers
  - Use Bigquery, a Google Cloud database for storing and querying
    massive datasets (there are equivalents on AWS)
  - Use Cloud Run, a Google Cloud managed compute platform for scaling
    containerised applications

These all have their own tradeoffs, from re-writing code for
compatibility reasons (and the additional burden of testing for
correctness against the original algorithm), to foregoing interactivity,
to operating system compatibility, to cost.

We’ll explore these in some detail over the coming days and weeks.

## A Workflow for Performant Code

It’s all too easy to focus on optimisng code before you really should.
That can be an incredible waste of time, and more than invalidate any
speedups you get from your optimisation efforts.

In general, the following steps will mostly prevent you from doing this:

  - Get the code working (we did that in the previous post)
  - Profile (we’ll do that shortly)
  - Fix obvious things, for instance pre-allocating large objects
    instead of growing them in a loop
  - Weigh options for optimising code, if required:
      - do nothing
      - implement `lapply`, or better, vectorisation (noting that it can
        consume a lot of memory)
      - Rcpp
      - parallelisation
      - bytecode compiler for small speedups
      - some combination of the above

## Warm-up: Profiling R Code

Profiling is the process of identifying bottlenecks in code.

Before we even think about optimising our code, we need to know what we
should be optimising. Profiling is the detective work that helps you
understand where your development time is best spent.

And while premature optimisation is to be avoided, performance *does*
matter.

Bottlenecks in your code might surprise you. We all want to be better
programmers - profiling code provides useful lessons and is an
opportunity to identify and fix bad practices.

So even if you delay optimising your code (you definitely should delay
it), there’s little overhead and lots to be gained from profiling your
code as you develop it.

And sometimes, you just have to, in order to fix a critical bottleneck.

In our case, we essentially know where the bottleneck is - it’s the
enormous dataframe of pairwise correlations that we try to compute in
memory. But still, we’ll profile the code to demonstrate how it’s done,
and to ensure we don’t have any surprises.

### First Step: Timing

Getting simple timings as a basic measure of performance is
straightforward.

  - `system.time()` is useful for timing blocks of code by running them
    once - but timing one evaluation can be misleading.
  - `Rprof()` can be used for timing execution of functions and
    statements.
  - `microbenchmark` is a de-facto standard among many R users. It
    provides statistical timing measurements and has some nice plot
    outputs. There’s also `rbenchmark`.

We’ll start by timing our code for calculating mean rolling pairwise
correlations using `microbenchmark`.

First we load our packages and data (you can get the data from our
GitHub repository - which if you clone will enable you to run the
relevant Rmd document directly). It consists of prices for SPX
constituents since 2015; we filter on a flag we added to indicate
whether a particular stock was in the index on a particular date:

``` r
library(tidyverse)
```

    ## -- Attaching packages ---------------------------------------------------------------------------------------- tidyverse 1.3.0 --

    ## v ggplot2 3.3.0     v purrr   0.3.3
    ## v tibble  2.1.3     v dplyr   0.8.3
    ## v tidyr   1.0.0     v stringr 1.4.0
    ## v readr   1.3.1     v forcats 0.4.0

    ## -- Conflicts ------------------------------------------------------------------------------------------- tidyverse_conflicts() --
    ## x dplyr::filter() masks stats::filter()
    ## x dplyr::lag()    masks stats::lag()

``` r
library(lubridate)
```

    ## 
    ## Attaching package: 'lubridate'

    ## The following object is masked from 'package:base':
    ## 
    ##     date

``` r
library(glue)
```

    ## 
    ## Attaching package: 'glue'

    ## The following object is masked from 'package:dplyr':
    ## 
    ##     collapse

``` r
library(here)
```

    ## here() starts at C:/Users/Kris/Documents/r-quant-recipes

    ## 
    ## Attaching package: 'here'

    ## The following object is masked from 'package:lubridate':
    ## 
    ##     here

``` r
library(microbenchmark)
library(profvis)
theme_set(theme_bw())

load(here::here("data", "spxprices_2015.RData"))
spx_prices <- spx_prices %>%
  filter(inSPX ==TRUE)
```

Next we “functionise” the steps in our pipeline of operations. This will
make profiling more straightforward, and the output of `microbenchmark`
easier to interpret:

``` r
# calculate returns to each stock
get_returns <- function(df) {
  df %>%
    group_by(ticker) %>%
    arrange(date, .by_group = TRUE) %>%
    mutate(return = close / dplyr::lag(close) - 1) %>%
    select(date, ticker, return)
}

# full join on date
fjoin_on_date <- function(df) {
  df %>%
    full_join(df, by = "date")
}

# ditch corr matrix diagonal, one half
wrangle_combos <- function(combinations_df) {
  combinations_df %>%
    ungroup() %>% 
  # drop diagonal 
    filter(ticker.x != ticker.y) %>% 
  # remove duplicate pairs (eg A-AAL, AAL-A)
    mutate(tickers = ifelse(ticker.x < ticker.y, glue("{ticker.x}, {ticker.y}"), glue("{ticker.y}, {ticker.x}"))) %>%
    distinct(date, tickers, .keep_all = TRUE) 
} 

pairwise_corrs <- function(combination_df, period) {
  combination_df %>%
    group_by(tickers) %>%
    arrange(date, .by_group = TRUE) %>%
    mutate(rollingcor = slider::slide2_dbl(
      .x = return.x, 
      .y = return.y, 
      .f = ~cor(.x, .y), 
      .before = period, 
      .complete = TRUE)
      ) %>%
    select(date, tickers, rollingcor)
  
} 

mean_pw_cors <- function(correlations_df) {
  correlations_df %>%
    group_by(date) %>%
    summarise(mean_pw_corr = mean(rollingcor, na.rm = TRUE))
} 
```

Now, let’s see what happens if we try to run the pipeline on the full
dataset:

``` r
returns_df <- get_returns(spx_prices)
combos_df <- fjoin_on_date(returns_df)
wrangled_combos_df <- wrangle_combos(combos_df)
corr_df <- pairwise_corrs(wrangled_combos_df, period = 60)
meancorr_df <- mean_pw_cors(corr_df)

# Error: cannot allocate vector of size 3.7 Gb
```

I can’t allocate enough RAM to hold one of the dataframes in memory.

I could change R’s memory allocation (by default on Windows I *think*
it’s 4GB) by doing `memory.limit(size = new_size)`, but from
experience I know that simply going large won’t solve this particular
problem, at least on my machine.

Now if we ask `microbenchmark` to time our code for us, we’ll get some
insights into where this breaks down.

Admittedly this is something of an awkward use-case - the typical
use-case is comparing the speed of different implementations of the same
thing, but here we’re using it to get insight into the timings of each
key step in our pipeline, to infer potential bottlenecks (which may or
may not be related to our memory issue, but it’s a decent starting
point).

As the process is quite long-running, we only run it twice (the default
is 100), measure the output in seconds, and only operate on a subset of
our data:

``` r
prices_subset <- spx_prices %>% 
  filter(date >= "2019-07-01", date < "2020-01-01")

mb <- microbenchmark(
  returns_df <- get_returns(prices_subset),
  combos_df <- fjoin_on_date(returns_df),
  wrangled_combos_df <- wrangle_combos(combos_df),
  corr_df <- pairwise_corrs(wrangled_combos_df, period = 60),
  meancorr_df <- mean_pw_cors(corr_df),
  times = 2,
  unit = "s",
  control = list(order = "block", warmup = 1)
)

mb
```

    ## Unit: seconds
    ##                                                        expr         min
    ##                    returns_df <- get_returns(prices_subset)   0.0510845
    ##                      combos_df <- fjoin_on_date(returns_df)   2.1008446
    ##             wrangled_combos_df <- wrangle_combos(combos_df)  39.6354533
    ##  corr_df <- pairwise_corrs(wrangled_combos_df, period = 60) 205.6261256
    ##                        meancorr_df <- mean_pw_cors(corr_df)   0.6687242
    ##           lq        mean      median          uq         max neval cld
    ##    0.0510845   0.0523182   0.0523182   0.0535519   0.0535519     2 a  
    ##    2.1008446   2.2839214   2.2839214   2.4669982   2.4669982     2 a  
    ##   39.6354533  39.8163891  39.8163891  39.9973250  39.9973250     2  b 
    ##  205.6261256 211.2760131 211.2760131 216.9259006 216.9259006     2   c
    ##    0.6687242   0.6930794   0.6930794   0.7174347   0.7174347     2 a

The bottleneck is quite obvious: it’s the operation that calculates the
rolling pairwise correlations. No surprise there.

If you call `boxplot` on the output of `microbenchmark`, you get a nice
graphical view of the results:

``` r
boxplot(mb, unit = "s", log = FALSE)
```

<img src="intro-profiling_files/figure-gfm/unnamed-chunk-6-1.png" style="display: block; margin: auto;" />

So now we’ve got some basic insight into which operations represent
bottlenecks in our pipeline. However, we often want to get more detailed
information. For instance, `microbenchmark` only tells us about time, it
doesn’t tells us about memory usage - and in this case, running out of
RAM is the important thing.

Looking into this requires a different profiling tool. `Rprof` ships
with base R, but recently I’ve been using `profvis`, which has a highly
interpretable graphical output.

## Using profvis

`profvis` is simple to use: just wrap an expression or function call in
`profvis({...})` and observe the HTML output, which opens in a new tab
in R Studio, and which you can save via the `prof_output` argument.

`profvis` can handle a block of code:

``` r
profvis({
  returns_df <- get_returns(prices_subset)
  combos_df <- fjoin_on_date(returns_df)
  wrangled_combos_df <- wrangle_combos(combos_df)
  corr_df <- pairwise_corrs(wrangled_combos_df, period = 60)
  meancorr_df <- mean_pw_cors(corr_df)
}, prof_output = 'profile_out.Rprof')
```

The graphical output shows the time spent on each line of code in
milliseconds. The graph is interactive - you can zoom and move around to
get better views.

You can also see each line of code and the memory allocated and
deallocated (negative values), and the time spent on each line:

![profiling
output](https://github.com/Robot-Wealth/r-quant-recipes/blob/master/performant-r/images/profvis_out.png)

We can see that the vast majority of time was spent in the
`pairwise_corrs` function, and that it allocated about 18.5GB of memory.

It’s no surprise that this function is our bottleneck, but seeing the
RAM usage quantified like that is certainly useful. Remember also that
we’re only using a small subset of our data here. Our actual problem is
much bigger.

So it’s quite clear that in order to solve this particular problem, we
need to find a way around R’s memory limitations with respect to the
`pairwise_corrs` operation.

## Conclusion

In this post we introduced the idea of scaling up our mean rolling
pairwise correlation operation to accommodate the constituents of the
S\&P 500. We listed some options for doing so, noting that they all
involve various trade-offs.

Basic profiling indicated that, as expected, the operation that performs
the pairwise rolling correlations is the bottleneck, allocating over 18
GB of RAM even on a small subset of the total problem.

Since R computations are by default carried out in-memory, we have a
problem. The following posts will explore various solutions.

Finally, we should take a look at the rolling pairwise correlations that
we calculated for the subset of our larger problem, since we really like
visualisation (and to check that things at least look superficially
sensible):

``` r
meancorr_df %>%
  na.omit() %>%
  ggplot(aes(x = date, y = mean_pw_corr)) +
    geom_line() +
    labs(
      x = "Date",
      y = "Mean Pairwise Correlation",
      title = "Rolling Mean Pairwise Correlation",
      subtitle = "SPX Constituents"
    )
```

<img src="intro-profiling_files/figure-gfm/unnamed-chunk-8-1.png" style="display: block; margin: auto;" />
