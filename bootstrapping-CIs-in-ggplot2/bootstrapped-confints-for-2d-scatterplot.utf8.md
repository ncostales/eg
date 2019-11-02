---
title: "flexibly plotting bootstrapped CIs on scatterplots with ggplot2"
output: github_document
---




Today I found a solution to a ggplotting problem that I've hit my head on enough times this past semester to make me sufficiently excited to write something up and share with whomever might see.

## the problem

As it typically goes, I'll have two measures, **x** and **y**, and I'd like to assess their correlation.
But these measures don't seem to be 'well-behaved'.
For example, they might look heteroskedastic:


```r
library(ggplot2)
set.seed(0)
n <- 100  ## n datapoints
x <- 1:n  ## some measure x
y <- rnorm(x, x, sd = x)  ## some measure y; mean and variance depends on x
p <- ggplot(data.frame(x, y), aes(x, y)) +
  geom_point(fill = "black", color = "white", shape = 21, size = 3) +
  theme(
    axis.ticks = element_blank(),
    axis.text  = element_blank(),
    axis.line  = element_blank(),
    panel.background = element_blank()
  )
p
```

<img src="bootstrapped-confints-for-2d-scatterplot_files/figure-gfm/x-and-y-1.png" style="display: block; margin: auto;" />

Being aware of the issues that have been raised regarding correlations and parametric stats (e.g., Wilcox & Rousselet, 2018; 10.1002/cpns.41), I'll be wary of trusting the p-values from parametric tests.
For example, the statistic I'd get from `cor.test(x, y)$p.value` is probably off the mark.

So I use an appropriate test, one that's robust to heteroskedasticity: percentile bootstrap of a confidence interval (CI) around the correlation statistic (see, e.g., Wilcox, Rousselet, & Pernet, 2018, 10.1080/00949655.2018.1501051).
For convenience, I wrap this test within the following function, `boot.bivar()`.
This function takes two vectors of the same length and spits out a bootstrapped p-value and 95% confidence interval of the linear correlation coefficient.


```r
boot.bivar <- function(x, y, n.resamples = 1000) {
  
  if (length(x) != length(y)) stop("x and y not same length")
  
  resample.matrix <- matrix(
    sample.int(length(x), length(x) * n.resamples, replace = TRUE),
    ncol = n.resamples
  )
  resamples <- apply(resample.matrix, 2, function(s) cor(x[s], y[s]))
  
  ## get p-value (alpha = 0.05, 2-tailed):
  prop.less.0 <- mean(resamples < 0)
  p <- 2 * min(prop.less.0, 1 - prop.less.0)
  
  ## get CI (95%):
  lb <- (1 / 20 * n.resamples) / 2
  ub <- n.resamples - lb
  resamples <- sort(resamples)
  
  return(c(p = p, ci95l = resamples[lb], ci95h = resamples[ub]))

}

boot.bivar(x, y)
```

```
##         p     ci95l     ci95h 
## 0.0000000 0.4478353 0.6989415
```

So that's what I'll use for inference. Great.

But now, I'd like to depict this uncertainty in my scatterplot.
Specifically, I'd like to overlay this CI as a layer in my plot.
Thinking within the ggplot universe, I first turned to `geom_smooth()`:


```r
p + geom_smooth(method = "lm")
```

<img src="bootstrapped-confints-for-2d-scatterplot_files/figure-gfm/unnamed-chunk-2-1.png" style="display: block; margin: auto;" />

**BUT.**
After thinking for a moment, and looking into `?geom_smooth`, I realize that `geom_smooth(method = "lm")` uses a parametric estimate of the CI.
It assumes things that aren't true of the data.

And although I have an appropriately calculated CI already in hand (from `boot.bivar()`), I also realize that `geom_smooth()` won't take as input a 'user-defined' CI.
So, my question has been: **how do I add a custom CI layer (e.g., from a bootstrapped CI) to a scatterplot in ggplot2?**

## a solution

Essentially the solution that I found was to use `predict.lm()` (and `sapply()`) to generate a bootstrapped CI around each predicted value, $\mathbf{\hat{y}} \sim \mathbf{x}\beta$, then to feed these values into `geom_ribbon()`, which draws the thing.
And to make this extensible, I wrapped this bootstrapping part within a function `boot.bivar.predict()`, which is similar to the one above.


```r
# function for getting confidence interval:
boot.bivar.predict <- function(
  .data, xname, yname, n.resamples = 1000, percent = 95) {
  
  .formula <- paste0(yname, " ~ ", xname)
  predictions <- sapply(
    seq_len(n.resamples),
    function(.)
      predict(
        lm(.formula, .data[sample.int(nrow(.data), replace = TRUE), ]),
        .data[xname],
      )
  )
  .alpha <- (100 - percent) / 100  ## convert to alpha level
  
  data.frame(
    ub = apply(predictions, 1, quantile, 1 - .alpha / 2),  ## 2-tailed
    lb = apply(predictions, 1, quantile, 0 + .alpha / 2)
  )
  
}
```

I then use it like this:


```r
xy.ci <- boot.bivar.predict(data.frame(x, y), "x", "y")  ## get prediction interval
p + geom_ribbon(aes(ymin = xy.ci$lb, ymax = xy.ci$ub), alpha = 0.3)  ## add to plot
```

<img src="bootstrapped-confints-for-2d-scatterplot_files/figure-gfm/unnamed-chunk-4-1.png" style="display: block; margin: auto;" />

## a more useful solution

**But, for a more useful solution, I added an additional step of embedding these two functions---bootstrapping and drawing via `geom_ribbon()`---within a ggplot2-style function, `stat_boot_ci()`.**
This additional step enables me to call `stat_boot_ci()` as if it were an out-of-the-box ggplot function, within a ggplot pipe: it inherits (e.g., I don't have to explicitly feed it `data`, `x`, or `y` args), and it takes arguments that `geom_ribbon()` would take (e.g., `alpha`).

To do this, I created `ggproto()` and `layer()` objects, embedded my bootstrapping function within, and set appropriate defaults.


```r
## embed function within ggplot framework:
BootCI <- ggproto(
  "BootCI", Stat,
  required_aes = c("x", "y"),

  compute_group = function(data, scales, params, n = 1000, percent = 95) {
    grid <- data.frame(x = data$x)

   predictions <- sapply(
      seq_len(n),
      function(.)
        predict(
          lm(y ~ x, data[sample.int(nrow(data), replace = TRUE), ]),
          grid,
        )
    )

    .alpha <- (100 - percent) / 200  ## 2 tailed
    grid$ymax <- apply(predictions, 1, quantile, 1 - .alpha)
    grid$ymin <- apply(predictions, 1, quantile, .alpha)

    grid

  }
)

stat_boot_ci <- function(mapping = NULL, data = NULL, geom = "ribbon",
                    position = "identity", na.rm = FALSE, show.legend = NA,
                    inherit.aes = TRUE, n = 1000, percent = 95, ...) {
  ## see: https://cran.r-project.org/web/packages/ggplot2/vignettes/extending-ggplot2.html
  layer(
    stat = BootCI, data = data, mapping = mapping, geom = geom,
    position = position, show.legend = show.legend, inherit.aes = inherit.aes,
    params = list(n = n, percent = percent, na.rm = na.rm, ...)
  )
}
```

I cribbed some of this from [here](https://cran.r-project.org/web/packages/ggplot2/vignettes/extending-ggplot2.html), which gives an illuminating walk-through of the guts of ggplot2.

To see our `stat_boot_ci()` function in its glory, I use it below, on our highly heteroskedastic data.
I compare the bootstrapped CI to a parametric one from `geom_smooth()`.
The `geom_smooth()` is displayed underneath, in red.


```r
p +
  geom_smooth(method = "lm", se = TRUE, fill = "red", alpha = 0.3) +
  stat_boot_ci(alpha = 0.3, n = 1E4, percent = 95)
```

<img src="bootstrapped-confints-for-2d-scatterplot_files/figure-gfm/unnamed-chunk-6-1.png" style="display: block; margin: auto;" />
Notice how the parametric CI underpredicts the variance when x is high, and overpredicts when x is low.


## update: a faster solution

Once I had specified the general form of this `stat_boot_ci()` function, I realized that it could be easily optimized.
In particular, the `lm()` function is [slow](https://rpubs.com/maechler/fast_lm) and provides many unneccesary things --- when in fact 
all we need to compute is $\mathbf{\hat y}$.
And while there are faster `lm` alternatives, both [in base](https://www.rdocumentation.org/packages/stats/versions/3.6.1/topics/lm.fit), and [in `RccpEigen`](https://www.rdocumentation.org/packages/RcppEigen/versions/0.3.3.5.0), even these light-weight functions are doing much more lifting than required.

The general problem is simple least squares:
\[\mathbf{\hat y} = \mathbf{x}b_1 + b_0\]
that is, where $\mathbf{x}$ is a column vector and the $b$s are scalars.
In particular, estimates of the variability in $\mathbf{\hat y}$s across the range of $\mathbf{x}$ is desired.
A bootstrapped distribution of $\mathbf{\hat y}$s, written as the asterisked $\mathbf{\hat y}^*$, is obtained by (1) resampling rows of $\mathbf{x}$ and $\mathbf{y}$ $N$ times, forming sets of $N$ resampled vectors $\mathbf{x}^*$ and $\mathbf{y}^*$, (2) estimating corresponding $b^*_0$ and $b^*_1$, (3) then applying each of these bootstrapped coefficients to the original $\mathbf{x}$:
\[\mathbf{\hat y}^* = \mathbf{x}b_1^* + b_0^*\]

Estimating $b^*$s can be done in a number of algebraically equivalent ways.
But, these ways are likely not eqivalent in terms of computational efficiency.

One method is terms of linear correlation and standard deviation:
\[b_1^* = \text{cor}(\mathbf{x}^*, \mathbf{y}^*) \tfrac{\text{sd}(\mathbf{y}^*)} {\text{sd}(\mathbf{x}^*)}\]
\[b_0^* = \bar y^* - \bar x^* b_1^*\]
where the bar indicates the mean. 
This formulation is as simple as it gets.

Another is in terms of linear algebra
<!-- \[\mathbf{b}^* = (\mathbf{X}^{*\text{T}}\mathbf{X}^*)^{-1}\mathbf{X}^{*\text{T}}\mathbf{y}^*\] -->
\[\mathbf{b}^* =  \mathbf{X}^{*\dagger} \mathbf{y}^*\]
where $\mathbf{b}^*$ is now a 2-dimensional column vector, $\mathbf{X}^*$ is a matrix with a column $\mathbf{x}^*$ and a column of all-ones, and $\dagger$ indicates the pseudo-inverse.
This approach would use matrix packages, and while these packages are generally fast, it feels a little like cutting a steak with a sword.
However, this approach could easily accommodate multiple regressors, should I desire to extend my plotting function to mulitple regression in the future. 
Also, the pseudoinverse does not need to be explicitly calculated; the above formulation is just one method of many.
For example, R can solve this system of linear equations via, well, `solve()`, or singular value decomposition can be used, as well.

I'm unsure which method might be the most efficient.

So, I use the microbenchmark package to compare the speed of these methods below, with code cribbed from [this page](https://www.alexejgossmann.com/benchmarking_r/).
This test also includes a check to make sure the methods yield equivalent $\mathbf{\hat y}$s.


```r
library(microbenchmark)
n <- 1000
x <- rnorm(n)
X <- cbind(rep(1, length(x)), x)  ## add intercept
y <- rnorm(n)

check_for_equal_coefs <- function(values) {
  tol <- 1e-12
  max_error <- max(
    c(
      abs(values[[1]] - values[[2]]),
      abs(values[[2]] - values[[3]]),
      abs(values[[1]] - values[[3]]),
      abs(values[[1]] - values[[4]]),
      abs(values[[1]] - values[[5]])
    )
  )
  max_error < tol
}

mbm <- microbenchmark(
  
  "lm" = {
    
    y.hat <- predict(lm(y ~ x))
    
    },
  
  "cor" = {
    
    b1 <- cor(x, y) * sd(y) / sd(x)
    b0 <- mean(y) - b1 * mean(x)
    y.hat <- b1 * x + b0
    
    },
  
  "svd" = {
    
    y.hat <- svd(X)$u %*% t(svd(X)$u) %*% y
    
    },
  
  "pinv" = {
    
    y.hat <- X %*% solve(t(X) %*% X) %*% t(X) %*% y

    },
  
  "lin.sys" = {
    
    y.hat <- c(X %*% solve(t(X) %*% X, t(X) %*% y))
    
    },
  
  check = check_for_equal_coefs
  
  )

mbm  ## lin.sys wins! (but cor close behind)
```

```
## Unit: microseconds
##     expr    min      lq      mean   median       uq     max neval cld
##       lm 2746.7 4488.70  6517.060  5141.60  8869.20 15886.2   100  b 
##      cor  111.1  241.30   414.801   335.25   418.65  3647.5   100 a  
##      svd 6390.0 8555.95 11666.334 10469.40 13151.95 80471.0   100   c
##     pinv 6256.6 8613.10 11466.749 10856.55 13742.85 24125.3   100   c
##  lin.sys  102.8  152.75   262.109   190.15   257.50  2495.0   100 a
```

```r
autoplot(mbm)
```

```
## Coordinate system already present. Adding new coordinate system, which will replace the existing one.
```

<img src="bootstrapped-confints-for-2d-scatterplot_files/figure-gfm/unnamed-chunk-7-1.png" style="display: block; margin: auto;" />


The correlation method and letting R `solve()` seem to be the fastest, with a slight advantage for the matrix approach.
This small advantage would be amplified with increasing numbers of iterations, so the `solve()` approach, although only slightly faster, might be preferable to `cor()`.
I verify this by embedding these methods within `sapply()` loops and running the microbenchmark on these loops.


```r
## test within sapply() loop ----

x <- rnorm(100)
y <- rnorm(100)
data <- data.frame(x = x, y = y)
grid <- data.frame(x = data$x)


## now test:

mbm <- microbenchmark(
  
  "cor" = {
    
    sapply(
      seq_len(1000),
      function(.) {
        s <- sample.int(length(grid$x), replace = TRUE)
        b1 <- cor(x[s], y[s]) * sd(y[s]) / sd(x[s])
        b0 <- mean(y[s]) - b1 * mean(x[s])
        b1 * x + b0
      }
    )
    
  },
  
  "lin.sys" = {
    
    sapply(
      seq_len(1000),
      function(.) {
        samp <- sample.int(length(grid$x), replace = TRUE)
        Xsamp <- X[samp, ]
        X %*% solve(t(Xsamp) %*% Xsamp, t(Xsamp) %*% y[samp])
      }
    )
    
    }
)

mbm
```

```
## Unit: milliseconds
##     expr      min       lq     mean   median       uq      max neval cld
##      cor 213.6561 228.6312 273.1775 248.1402 310.4256 477.8796   100   b
##  lin.sys 105.4995 129.3700 180.7424 141.0149 214.4464 430.6556   100  a
```

```r
autoplot(mbm)
```

```
## Coordinate system already present. Adding new coordinate system, which will replace the existing one.
```

<img src="bootstrapped-confints-for-2d-scatterplot_files/figure-gfm/unnamed-chunk-8-1.png" style="display: block; margin: auto;" />
`solve()` it is.


## the final function

```r
BootCI <- ggplot2::ggproto(
  "BootCI", ggplot2::Stat, 
  required_aes = c("x", "y"),
  compute_group = function(data, scales, params, n = 1000, percent = 95) {
    
    grid <- data.frame(x = data$x)
    X <- cbind(rep(1, length(x)), x)  ## design matrix (includes intercept)
    
    predictions <- sapply(
      seq_len(n),
      function(.) {
        samp <- sample.int(length(grid$x), replace = TRUE)
        Xsamp <- X[samp, ]
        X %*% solve(t(Xsamp) %*% Xsamp, t(Xsamp) %*% y[samp])  ## get bs then dot with X
      }
    )
    
    .alpha <- (100 - percent) / 200  ## 2 tailed
    grid$ymax <- apply(predictions, 1, quantile, 1 - .alpha)
    grid$ymin <- apply(predictions, 1, quantile, .alpha)
    
    grid
    
  }
)

stat_boot_ci <- function(mapping = NULL, data = NULL, geom = "ribbon",
                         position = "identity", na.rm = FALSE, show.legend = NA, 
                         inherit.aes = TRUE, n = 1000, percent = 95, ...) {
  ## see: https://cran.r-project.org/web/packages/ggplot2/vignettes/extending-ggplot2.html
  
  ggplot2::layer(
    stat = BootCI, data = data, mapping = mapping, geom = geom, 
    position = position, show.legend = show.legend, inherit.aes = inherit.aes,
    params = list(n = n, percent = percent, na.rm = na.rm, ...)
  )
  
}
```