


# Functionals 

<!--  
  library(pryr)
  library(stringr)
  find_funs("package:base", fun_calls, fixed("match.fun"))
  find_funs("package:base", fun_args, ignore.case("^(fun|f)$"))
-->

## Introduction

"To become significantly more reliable, code must become more transparent. In particular, nested conditions and loops must be viewed with great suspicion. Complicated control flows confuse programmers. Messy code often hides bugs."
--- [Bjarne Stroustrup](http://www.stroustrup.com/Software-for-infrastructure.pdf)

Higher-order functions encompass any functions that either take a function as an input or return a function as output. We've already seen closures, functions returned by another function. The complement to a closure is a __functional__, a function that takes a function as an input and returns a vector as output. 

Here's a simple functional, it takes an input function and calls it with some random input:


```r
randomise <- function(f) f(runif(1e3))
randomise(mean)
# [1] 0.503
randomise(sum)
# [1] 494
```


This function is not terribly useful, but it illustrates the basic idea: since functions are first class objects in R, there's no difference between calling a function with a vector or function as input. The chances are that you've already used a functional: the most frequently used are `lapply()`, `apply()` and `tapply()`. These three functions all take a function as input (among other things) and give a vector as output.

Many functionals (like `lapply()`) offer alternatives to for loops. For loops have a bad rap in R, and some programmers try to eliminate them at all costs. The performance story is a little more complicated than what you might have heard (we'll explore that in the [[performance]] chapter); the real downside of for loops is that they're not very expressive. A for loop conveys that you're iterating over something, but it doesn't communicate the higher-level task you're trying to complete. Functionals are not as general as for loops, but by being more specific they allow you to communicate more clearly. A functional allows you to say I want to transform each element of this list, or each row of this array.

As well as more clearly communicating intent, functionals reduce the chances of bugs, and can be more efficient. Both of these features occur because functionals are used by many people, so they will be well tested, and may have been implemented with an eye to performance. For example, many functionals in base R are written in C, and often use a few tricks to get extra performance.

As well as replacements for for loops, functionals do play other roles. They are also useful tools for encapsulating common data manipulation tasks, the split-apply-combine pattern; for thinking "functionally"; and for working with mathematical functions. In this chapter, you'll learn about:

* Functionals that replace a common pattern of for-loop use, like `lapply`, `vapply` and `Map`.

* Functionals for manipulating common R data structures, like `apply`, `split`, `tapply` and the plyr package.

* Popular functionals from other programming languages, like `Map`, `Reduce` and `Filter`.

* Mathematical functionals, like `integrate`, `uniroot`, and `optim`.

We'll also talk about how (and why) you might convert loop to use a functional. The chapter concludes with a case study where we take simple scalar addition and use functionals to build a complete family of addition functions including vectorised addition, sum, cumulative sum, and row- and column-wise summation. 

The focus in this chapter is on clear communication with your code, and developing tools to solve wide classes of problems. This will not always produce the fastest code, but it is a mistake to focus on speed until you know it will be a problem. Once you do have clear, correct code you can make it fast using the techniques in the [[performance]] chapter.

## My first functional: `lapply()`

The simplest functional is `lapply()`, which you may already be familiar with. `lapply()` takes a function and applies it to each element of a list, saving the results back into a list.  `lapply()` is the building block for many other functionals, so it's important to understand how it works

![A sketch of how `lapply()` works](diagrams/lapply.png)

`lapply()` is written in C for performance, but we can create a simple R implementation that works the same way:


```r
lapply2 <- function(x, f, ...) {
  out <- vector("list", length(x))
  for (i in seq_along(x)) {
    out[[i]] <- f(x[[i]], ...)
  }
  out
}
```


From this code, you can see that `lapply()` is a wrapper around a common for loop pattern: we create a space for output, and then fill it in, applying `f()` to each component of the list. All other for loop functionals build on this base, modifying either the input, the output, or what data the function is applied to. From this code you can see that `lapply()` will also works with vectors: both `length()` and `'[[` work the same way for lists and vectors.

`lapply()` makes it easier to work with lists by eliminating much of the boilerplate, focussing on the operation you're applying to each piece:


```r
# Create some random data
l <- replicate(20, runif(sample(1:10, 1)), simplify = FALSE)

# With a for loop
out <- vector("list", length(l))
for (i in seq_along(l)) {
  out[[i]] <- length(l[[i]])    
}
unlist(out)
#  [1]  2  7  2  3  2  7  8 10  8  2  2  3  6 10  5  7  4  4 10  3

# With lapply
unlist(lapply(l, length))
#  [1]  2  7  2  3  2  7  8 10  8  2  2  3  6 10  5  7  4  4 10  3
```


(We're using `unlist()` to convert the output from a list to a vector to make the output more compact. We'll see other ways of making the output a vector shortly.)

Since data frames are also lists, `lapply()` is useful when you want to do something to each column of a data frame:


```r
# What class is each column?
unlist(lapply(mtcars, class))
#       mpg       cyl      disp        hp      drat        wt      qsec 
# "numeric" "numeric" "numeric" "numeric" "numeric" "numeric" "numeric" 
#        vs        am      gear      carb 
# "numeric"  "factor" "numeric" "numeric"

# Divide each column by the mean
mtcars[] <- lapply(mtcars, function(x) x / mean(x))
# Warning: argument is not numeric or logical: returning NA
# Warning: / not meaningful for factors
```


The pieces of `x` are always supplied as the first argument to `f`. You can override this using R's regular function calling semantics, supplying additional named arguments. For example, imagine you wanted to compute various trimmed means of the same dataset. `trim` is the second parameter of `mean()`, so we want to vary that, keeping the first argument (`x`) fixed.  It's easy provided that you remember that the following two calls are equivalent 


```r
mean(1:100, trim = 0.1)
# [1] 50.5
mean(0.1, x = 1:100)
# [1] 50.5
```


So to use `lapply()` with the second argument, we just need to name the first argument:


```r
trims <- c(0, 0.1, 0.2, 0.5)
x <- rcauchy(100)
unlist(lapply(trims, mean, x = x))
# [1] 12.1618 -0.0911 -0.0886  0.0182
```


### Looping patterns

When using `lapply()` and friends, it's useful to remember that there are usually three ways to loop over an vector: 

1. loop over the elements of the vector: `for(x in xs)`
2. loop over the numeric indices of the vector: `for(i in seq_along(xs))`
3. loop over the names of the vector: `for(nm in names(xs))`

If you're saving the results from a for loop, you usually can't use the first form because it makes very inefficient code.  When extending an existing data structure, all the existing data must be copied every time you extend it:


```r
xs <- runif(1e3)
res <- c()
for(x in xs) {
  # This is slow!
  res <- c(res, sqrt(x))
}
```


It's much better to create enough space for the output and then fill it in, using the second looping form:


```r
res <- numeric(length(xs))
for(i in seq_along(xs)) {
  res[i] <- sqrt(xs[i])
}
```


Corresponding to the three ways to use a for loop there are three ways to use `lapply()` with an object:


```r
lapply(xs, function(x) {})
lapply(seq_along(xs), function(i) {})
lapply(names(xs), function(nm) {})
```


Typically you use the first form because `lapply()` takes care of saving the output for you. However, if you need to know the position or the name of the element you're working with, you'll need to use the second or third form; they give you both the position of the object (`i`, `nm`) and its value (`xs[[i]]`, `xs[[nm]]`). If you're struggling to solve a problem using one form, you might find it easier with a different form.

If you're working with a list of functions, remember to use `call_fun`:


```r
call_fun <- function(f, ...) f(...)
f <- list(sum, mean, median, sd)
lapply(f, call_fun, x = runif(1e3))
# [[1]]
# [1] 498
# 
# [[2]]
# [1] 0.498
# 
# [[3]]
# [1] 0.49
# 
# [[4]]
# [1] 0.29
```


Or you could create a variant, `fapply()`, specifically for working with lists of functions:


```r
fapply <- function(fs, ...) {
  out <- vector("list", length(fs))
  for (i in seq_along(fs)) {
    out[[i]] <- fs[[i]](...)
  }
  out
}
fapply(f, x = runif(1e3))
# [[1]]
# [1] 490
# 
# [[2]]
# [1] 0.49
# 
# [[3]]
# [1] 0.48
# 
# [[4]]
# [1] 0.292
```



### Exercises

* The function `scale01()` given below scales a vector to have range 0-1. How would you apply it to every column in a data frame? How would you apply it to every numeric column in a data frame?

    
    ```r
    scale01 <- function(x) {
      rng <- range(x, na.rm = TRUE)
      (x - rng[1]) / (rng[2] - rng[1])
    }
    ```


* For each formula in the list below, use a for-loop and lapply to fit the corresponding model to the `mtcars` dataset

    
    ```r
    formulas <- list(
      mpg ~ disp,
      mpg ~ I(1 / disp),
      mpg ~ disp + wt,
      mpg ~ I(1 / disp) + wt
    )
    ```


* Fit the model `mpg ~ disp` to each of the bootstrap replicates of `mtcars` in the list below, using a for loop and then `lapply()`. Can you do it without an anonymous function?

    
    ```r
    bootstraps <- lapply(1:10, function(i) {
      rows <- sample(1:nrow(mtcars), rep = TRUE)
      mtcars[rows, ]
    })
    ```


* For each model in the previous two exercises extract the R^2 using the function below.

  
  ```r
  rsq <- function(mod) summary(mod)$r.squared
  ```


## For loop functionals: friends of `lapply()`

The art of using functionals is to recognise what common looping patterns are implemented in existing base functionals, and then use them instead of loops. Once you've mastered the existing functionals, the next step is to start writing your own: if you discover you're duplicating the same looping pattern in many places, you should extract it out into its own function. 

The following sections build on `lapply()` and discuss:

* `sapply()` and `vapply()`, variants of `lapply()` that produce 
vectors, matrices and arrays as __output__, instead of lists.

* `Map()` and `mapply()` which iterate over multiple __input__ data structures in parallel.

* __Parallel__ versions of `lapply()` and `Map()`, `mclapply()` and `mcMap()`

* __Rolling computations__, showing how a new problem can be solved with for loops, or by building on top of `lapply()`.

### Vector output: `sapply` and `vapply`

`sapply()` and `vapply()` are very similar to `lapply()` except they will simplify their output to produce an atomic vector. `sapply()` guesses, while `vapply()` takes an additional argument specifying the output type. `sapply()` is useful for interactive use because it saves typing, but if you use it inside your functions you will get weird errors if you supply the wrong type of input. `vapply()` is more verbose, but gives more informative errors messages and never fails silently, so is better suited for use inside other functions.

The following example illustrates these differences.  When given a data frame `sapply()` and `vapply()` give the same results. When given an empty list, `sapply()` has no basis to guess the correct type of output, and returns `NULL`, instead of the more correct zero-length logical vector.


```r
sapply(mtcars, is.numeric)
#   mpg   cyl  disp    hp  drat    wt  qsec    vs    am  gear  carb 
#  TRUE  TRUE  TRUE  TRUE  TRUE  TRUE  TRUE  TRUE FALSE  TRUE  TRUE
vapply(mtcars, is.numeric, logical(1))
#   mpg   cyl  disp    hp  drat    wt  qsec    vs    am  gear  carb 
#  TRUE  TRUE  TRUE  TRUE  TRUE  TRUE  TRUE  TRUE FALSE  TRUE  TRUE
sapply(list(), is.numeric)
# list()
vapply(list(), is.numeric, logical(1))
# logical(0)
```


If the function returns results of different types or lengths, `sapply()` will silently return a list, while `vapply()` will throw an error. `sapply()` is fine for interactive use because you'll normally notice if something went wrong, but it's dangerous when writing functions. 

The following example illustrates a possible problem when extracting the class of columns in data frame: if you falsely assume that class only has one value and use `sapply()` you won't find out about the problem until some future function is given a list instead of a character vector.


```r
df <- data.frame(x = 1:10, y = letters[1:10])
sapply(df, class)
#         x         y 
# "integer"  "factor"
vapply(df, class, character(1))
#         x         y 
# "integer"  "factor"

df2 <- data.frame(x = 1:10, y = Sys.time() + 1:10)
sapply(df2, class)
# $x
# [1] "integer"
# 
# $y
# [1] "POSIXct" "POSIXt"
vapply(df2, class, character(1))
# Error: values must be length 1, but FUN(X[[2]]) result is length 2
```


`sapply()` is a thin wrapper around `lapply()`, transforming a list into a vector in the final step; `vapply()` reimplements `lapply()` but assigns results into a vector (or matrix) of the appropriate type instead of into a list. The following code shows pure R implementation of the essence of `sapply()` and `vapply()`; the real functions have better error handling and preserve names, among other things. 


```r
sapply2 <- function(x, f, ...) {
  res <- lapply2(x, f, ...)
  simplify2array(res)
}

vapply2 <- function(x, f, f.value, ...) {
  out <- matrix(rep(f.value, length(x)), nrow = length(x))
  for (i in seq_along(x)) {
    res <- f(x[i], ...)
    stopifnot(
      length(res) == length(f.value), 
      typeof(res) == typeof(f.value)
    )
    out[i, ] <- res
  }
  out
}
vapply2(1:10, f, logical(1))
# Error: could not find function "f"
```


![Schematics of `sapply` and `vapply`, cf `lapply`.](diagrams/sapply-vapply.png)

`vapply()` and `sapply()` are like `lapply()`, but with different outputs; the following section discusses `Map()`, which is like `lapply()` but with different inputs. 

### Multiple inputs: `Map` (and `mapply`)

With `lapply()`, only one argument to the function varies; the others are fixed. This makes it poorly suited for some problems. For example, how would you find the weighted means when you have two lists, one of observations and the other of weights:


```r
# Generate some sample data
xs <- replicate(10, runif(10), simplify = FALSE)
ws <- replicate(10, rpois(10, 5) + 1, simplify = FALSE)
```


It's easy to use `lapply()` to compute the unweighted means:


```r
unlist(lapply(xs, mean))
#  [1] 0.662 0.689 0.548 0.523 0.591 0.428 0.340 0.515 0.479 0.473
```


But how could we supply the weights to `weighted.mean()`? `lapply(x, means, w)` won't work because the additional arguments to `lapply()` are passed to every call. We could change looping forms:


```r
unlist(lapply(seq_along(xs), function(i) {
  weighted.mean(xs[[i]], ws[[i]])
}))
#  [1] 0.670 0.687 0.536 0.522 0.612 0.424 0.354 0.500 0.502 0.443
```


This works, but is a little clumsy. A cleaner alternative is to use `Map`, a variant of `lapply()`, where all arguments vary.  This lets us write:


```r
unlist(Map(weighted.mean, xs, ws))
#  [1] 0.670 0.687 0.536 0.522 0.612 0.424 0.354 0.500 0.502 0.443
```


(Note that the order of arguments is a little different: with `Map()` the function is the first argument, with `lapply()` it's the second.

This is equivalent to:


```r
stopifnot(length(x) == length(w))
out <- vector("list", length(x))
for (i in seq_along(x)) {
  out[[i]] <- weighted.mean(x[[i]], w[[i]])
}
```


There's a natural equivalence between `Map()` and `lapply()` because you can always convert a `Map()` to an `lapply()` that iterates over indices, but using `Map()` is more concise, and more clearly indicates what you're trying to do.

`Map` is useful whenever you have two (or more) lists (or data frames) that you need to process in parallel. For example, another way of standardising columns, is to first compute the means and then divide by them. We could do this with `lapply()`, but if we do it in two steps, we can more easily check the results at each step, which is particularly important if the first step is more complicated.


```r
mtmeans <- lapply(mtcars, mean)
mtmeans[] <- Map(`/`, mtcars, mtmeans)

# In this case, equivalent to
mtcars[] <- lapply(mtcars, function(x) x / mean(x))
```


If some of the arguments should be fixed, and not varying, you need to use an anonymous function:


```r
Map(function(x, w) weighted.mean(x, w, na.rm = TRUE), xs, ws)
```


We'll see a more compact way to express the same idea in the next chapter.

<!-- This should be a sidebar -->

You may be more familiar with `mapply()` than `Map()`. I prefer `Map()` because:

* it is equivalent to `mapply` with `simplify = FALSE`, which is almost always what you want. 

* Instead of using an anonymous function to provide constant inputs, `mapply` has the `MoreArgs` argument which takes a list of extra arguments that will be supplied, as is, to each call. This breaks R's usual lazy evaluation semantics, and is inconsistent with other functions.

In brief, `mapply()` is more complicated for little gain.

### Rolling computations

What if you need a for-loop replacement that doesn't exist in base R? You can often create your own by recognising common looping structures and implementing your own wrapper. For example, you might be interested in smoothing your data using a rolling (or running) mean function:


```r
rollmean <- function(x, n) {
  out <- rep(NA, length(x))

  offset <- trunc(n / 2)
  for (i in (offset + 1):(length(x) - n + offset - 1)) {
    out[i] <- mean(x[(i - offset):(i + offset - 1)])
  }
  out
}
x <- seq(1, 3, length = 1e2) + runif(1e2)
plot(x)
lines(rollmean(x, 5), col = "blue", lwd = 2)
lines(rollmean(x, 10), col = "red", lwd = 2)
```

![plot of chunk roll-mean](figure/roll-mean.png) 


But if the noise was more variable (i.e. it had a longer tail) you might worry that your rolling mean was too sensitive to the occassional outlier and instead implement a rolling median. 


```r
x <- seq(1, 3, length = 1e2) + rt(1e2, df = 2) / 3
plot(x)
lines(rollmean(x, 5), col = "red", lwd = 2)
```

![plot of chunk outliers](figure/outliers.png) 


To modify `rollmean()` to `rollmedian()` all you need to do is replace `mean` with `median` inside the loop, but instead of copying and pasting to create a new function, we could extract the idea of computing a rolling summary into its own function:


```r
rollapply <- function(x, n, f, ...) {
  out <- rep(NA, length(x))

  offset <- trunc(n / 2)
  for (i in (offset + 1):(length(x) - n + offset - 1)) {
    out[i] <- f(x[(i - offset):(i + offset - 1)], ...)
  }
  out
}
plot(x)
lines(rollapply(x, 5, median), col = "red", lwd = 2)
```

![plot of chunk roll-apply](figure/roll-apply.png) 


You might notice that the internal loop looks pretty similar to a `vapply()` loop, so we could rewrite the function as:


```r
rollapply <- function(x, n, f, ...) {
  offset <- trunc(n / 2)
  locs <- (offset + 1):(length(x) - n + offset - 1)
  vapply(locs, function(i) f(x[(i - offset):(i + offset - 1)], ...),
    numeric(1))
}
```


This is effectively the same as the implementation in `zoo::rollapply()`, but it provides many more features and much more error checking.

### Parallelisation

One thing that's interesting about the defintions of `lapply()` is that because each iteration is isolated from all others, the order in which they are computed doesn't matter. For example, while `lapply3()`, defined below, scrambles the order in which computation occurs, the results are same every time:


```r
lapply3 <- function(x, f, ...) {
  out <- vector("list", length(x))
  for (i in sample(seq_along(x))) {
    out[[i]] <- f(x[[i]], ...)
  }
  out
}
unlist(lapply3(1:10, sqrt))
#  [1] 1.00 1.41 1.73 2.00 2.24 2.45 2.65 2.83 3.00 3.16
unlist(lapply3(1:10, sqrt))
#  [1] 1.00 1.41 1.73 2.00 2.24 2.45 2.65 2.83 3.00 3.16
```


This has a very important consequence: since we can compute each element in any order, it's easy to dispatch the tasks to different cores, and compute in parallel.  This is what `mclapply()` (and `mcMap`) in the parallel package do:


```r
library(parallel)
unlist(mclapply(1:10, sqrt, mc.cores = 4))
#  [1] 1.00 1.41 1.73 2.00 2.24 2.45 2.65 2.83 3.00 3.16
```


In this case `mclapply()` is actually slower than `lapply()`, becuase the cost of the individual computations is low, and some additional work is needed to send the computation to the different cores then collect the results together. If we take a more realistic example, generating bootstrap replicates of a linear model, we see more of an advantage:


```r
boot_df <- function(x) x[sample(nrow(x), rep = T), ]
rsquared <- function(mod) summary(mod)$r.square
boot_lm <- function(i) {
  rsquared(lm(mpg ~ wt + disp, data = boot_df(mtcars)))
}

system.time(lapply(1:500, boot_lm))
#    user  system elapsed 
#   0.766   0.007   0.773
system.time(mclapply(1:500, boot_lm, mc.cores = 2))
#    user  system elapsed 
#   0.003   0.006   0.419
```


It is rare to get an exactly linear improvement with increasing number of cores, but if your code uses `lapply()` or `Map()`, this is an easy way to improve performance.

### Exercises

* Use `vapply()` to:

  * Compute the standard deviation of every column in a numeric data frame.
  
  * Compute the standard deviation of of every numeric column in a mixed data frame (Hint: you'll need to use `vapply()` twice)

* Recall: why is using `sapply()` to get the `class()` of each element in a data frame dangerous?

* The following code simulates the performance of a t-test for non-normal data. Use `sapply()` and an anonymous function to extract the p value from every trial. Extra challenge: get rid of the anonymous function and use the `'[[` function.  

    
    ```r
    trials <- replicate(100, t.test(rpois(10, 10), rpois(7, 10)), 
      simplify = FALSE)
    ```


* Implement a combination of `Map()` and `vapply()` to create an `lapply()` variant that iterates in parallel over all of its inputs and stores its outputs in a vector (or a matrix).  What arguments should the function take?

* What does `replicate()` do? What sort of for loop does it eliminate? Why do its arguments differ from `lapply()` and friends?

* Implement `mcsapply()`, a multicore version of `sapply()`.  Can you implement `mcvapply()` a parallel version of `vapply()`? Why/why not?

* Implement a version of `lapply()` that supplies `f()` with both the name and the value of each component.

## Data structure functionals

As well as functionals that exist to eliminate common looping constructs, another family of functionals works to eliminate loops for common data manipulation tasks. In this section, we'll give a brief overview of the available options. We'll show you some of the available options, hint at how they can help you, and point you in the right direction to learn more. We'll cover three categories of data structure functionals:

* base functions for working with matrices: `apply()`, `sweep()` and `outer()`

* `tapply()`, which summarises a vector divided into groups by the values of another vector

* the `plyr` package, which generalises the ideas of `tapply()` to work with inputs of data frames, lists and arrays, and outputs of data frames, lists, arrays and nothing

### Matrix and array operations

So far, all the functionals we've seen work with 1d input structures. The three functionals in this section provide useful tools for working with high-dimensional data strucures.  `apply()` is a variant of `sapply()` that works with matrices and arrays. You can think of it as an operation that summarises a matrix or array, collapsing each row or column to a single number.  It has four arguments: 

* `X`, the matrix or array to summarise
* `MARGIN`, an integer vector giving the dimensions to summarise over, 1 = rows, 2 = columns, etc
* `FUN`, a summary function
* `...` other arguments passed on to `FUN`

A typical example of `apply()` looks like this


```r
a <- matrix(1:20, nrow = 5)
apply(a, 1, mean)
# [1]  8.5  9.5 10.5 11.5 12.5
apply(a, 2, mean)
# [1]  3  8 13 18
```


There are a few caveats to using `apply()`: it does not have a simplify argument, so you can never be completely sure what type of output you will get. This generally means that `apply()` is not safe to use inside a function, unless you carefully check the inputs. `apply()` is also not idempotent in the sense that if the summary function is the identity operator, the output is not always the same as the input:


```r
a1 <- apply(a, 1, identity)
identical(a, a1)
# [1] FALSE
identical(a, t(a1))
# [1] TRUE
a2 <- apply(a, 2, identity)
identical(a, a2)
# [1] TRUE
```


(You can put high-dimensional arrays back in the right order using `aperm()`, or use `plyr::aaply()`, which is idempotent.)

`sweep()` is a function that allows you to "sweep" out the values of a summary statistic. It is most often useful in conjunction with `apply()` and it often used to standardise arrays in some way. The following example scales a matrix so that all values lie between 0 and 1.


```r
x <- matrix(runif(20), nrow = 4)
x1 <- sweep(x, 1, apply(x, 1, min))
x2 <- sweep(x1, 1, apply(x1, 1, max), "/")
```


The final matrix functional is `outer()`. It's a little different in that it takes multiple vector inputs and creates a matrix or array output where the input function is run over every combination of the inputs:


```r
# Create a times table
outer(1:9, 1:9, "*")
#       [,1] [,2] [,3] [,4] [,5] [,6] [,7] [,8] [,9]
#  [1,]    1    2    3    4    5    6    7    8    9
#  [2,]    2    4    6    8   10   12   14   16   18
#  [3,]    3    6    9   12   15   18   21   24   27
#  [4,]    4    8   12   16   20   24   28   32   36
#  [5,]    5   10   15   20   25   30   35   40   45
#  [6,]    6   12   18   24   30   36   42   48   54
#  [7,]    7   14   21   28   35   42   49   56   63
#  [8,]    8   16   24   32   40   48   56   64   72
#  [9,]    9   18   27   36   45   54   63   72   81
```


Good places to learn more about `apply()` and friends are:

* [Using apply, sapply, lapply in R](http://petewerner.blogspot.com/2012/12/using-apply-sapply-lapply-in-r.html) by Peter Werner.
* [The infamous apply function](http://rforpublichealth.blogspot.no/2012/09/the-infamous-apply-function.html) by Slawa Rokicki.
* [The R apply function – a tutorial with examples](http://forgetfulfunctor.blogspot.com/2011/07/r-apply-function-tutorial-with-examples.html) by axiomOfChoice.
* The stack overflow question ["R Grouping functions: sapply vs. lapply vs. apply. vs. tapply vs. by vs. aggregate vs"](http://stackoverflow.com/questions/3505701).

### Group apply

You can think about `tapply()` as a generalisation to `apply()` that allows for "ragged" arrays, where each row can have different numbers of rows. This is often needed when you're trying to summarise a data set. For example, imagine you've collected some pulse rate from a medical trial, and you want to compare the two groups:


```r
pulse <- round(rnorm(22, 70, 10 / 3)) + rep(c(0, 5), c(10, 12))
group <- rep(c("A", "B"), c(10, 12))

tapply(pulse, group, length)
#  A  B 
# 10 12
tapply(pulse, group, mean)
#    A    B 
# 68.8 75.5
```


It's easiest to understand how `tapply()` works by first creating a "ragged" data structure from the inputs. This is the job of the `split()` function, which takes two inputs and returns a list, where all the elements in the first vector with equal entries in the second vector get put in the same element of the list:


```r
split(pulse, group)
# $A
#  [1] 65 67 73 66 71 66 76 70 69 65
# 
# $B
#  [1] 75 71 74 76 68 79 78 78 80 76 75 76
```


Then you can see that `tapply()` is just the combination of `split()` and `sapply()`:


```r
tapply2 <- function(x, group, f, ..., simplify = TRUE) {
  pieces <- split(x, group)
  sapply(pieces, f, simplify = simplify)
}
tapply2(pulse, group, length)
#  A  B 
# 10 12
tapply2(pulse, group, mean)
#    A    B 
# 68.8 75.5
```


Be able to rewrite our `tapply()` as a combination of `split()` and `sapply()` is a good indication that we've been able to extract independent pieces that we can recombine to solve new problems.

### The plyr package

One challenge with using the base functionals is that they have grown organically over time, and have been written by multiple authors. This means that they are not very consistent. For example,

* The simplify argument is called `simplify` in `tapply()` and `sapply()`, but `SIMPLIFY` for `mapply()`, and `apply()` lacks the argument altogether.

* `vapply()` is a variant of `sapply()` that allows you to describe what the output should be, but there are no corresponding variants of `tapply()`, `apply()`, or `Map()`.

* The first to most functionals is the vector, but the first argument to `Map()` is the function.

This makes learning these operators challenging, as you have to memorise all of the variations. Additionally, if you think about the combination of input and output types, base R only provides a partial set of functions:

|            | list   | data frame | array  |
|------------|--------|------------|--------|
| list       | lapply |            | sapply |
| data frame | by     |            |        |
| array      |        |            | apply  |

This was one of the driving forces behind the creation of the plyr package, which provides consistently named functions with consistently named arguments and implements all combinations of input and output data structures:

|            | list   | data frame | array | 
|------------|--------|------------|-------|
| list       | llply  | ldply      | laply |
| data frame | dlply  | ddply      | daply |
| array      | alply  | adply      | aaply |

Each of these functions splits up the input, applies a function to each piece and then joins the results back together. Overall, this process is called "split-apply-combine", and you can read more about it and plyr in [The Split-Apply-Combine Strategy for Data Analysis](http://www.jstatsoft.org/v40/i01/), an open-access article published in the Journal of Statistical Software.

### Exercises

* How does `apply()` arrange the output.  Read the documentation and perform some experiments.

* There's no equivalent to `split()` + `vapply()`. Should there be? When would it be useful? Implement it yourself.

* Implement a pure R version of `split()`. (Hint: use unique and subseting)

* What other types of input and output are missing? Brainstorm before you look up some answers in the [plyr paper](http://www.jstatsoft.org/v40/i01/)

## Functional programming

<!-- 
  http://www.haskell.org/ghc/docs/latest/html/libraries/base/Prelude.html
  http://docs.scala-lang.org/overviews/collections/trait-traversable.html#operations_in_class_traversable

  Clojure and python documentation is not so useful
 -->

Another way of thinking about functionals is as a set of general tools for altering, subsetting and collapsing lists. Every functional programming has three tools for this: `Map()`, `Reduce()`, and `Filter()`. We've seen `Map()` already, and the following sections described `Reduce()`, a powerful tool for extending two-argument functions, and `Filter()`, a member of an important class of functionals that work with predicates, functions that return a single boolean.

### `Reduce()`

`Reduce()` recursively reduces a vector, `x`, to a single value by recursively calling a function `f` with two arguments at a time.  It combines the first two elements with `f`, then combines the result of that call with the third element, and so on. Reduce is also known as fold, because it folds together adjacent elements in the list.

The following two examples show what `Reduce` does with an infix and prefix function:


```r
Reduce(`+`, 1:3)
((1 + 2) + 3)

Reduce(sum, 1:3)
sum(sum(1, 2), 3)
```


As you might have come to expect by now, the essence of `Reduce()` can be described by a simple for loop: 


```r
Reduce2 <- function(f, x) {
  out <- x[[1]]
  for(i in seq(2, length(x))) {
    out <- f(out, x[[i]])
  }
  out  
}
```


The real `Reduce()` is more complicated because it includes arguments to control whether the values are reduced from the left or from the right (`right`), an optional initial value (`init`), and an option to output every intermediate result (`accumulate`).

Reduce is an elegant way of turning binary functions into functions that can deal with any number of arguments. It's useful for implementing many types of recursive operations, like merges and intersections (We'll see another use in the final case study). For example, imagine you had a list of numeric vectors, and you wanted to find the values that occured in every element:


```r
l <- replicate(5, sample(1:10, 15, rep = T), simplify = FALSE)
l
# [[1]]
#  [1]  6  3  6  1  9  1  5  9  5  8  4  1  3 10  5
# 
# [[2]]
#  [1] 4 9 8 1 2 5 8 4 3 5 4 3 6 5 3
# 
# [[3]]
#  [1]  7  3  8 10 10  2  2  3  4  9  3  5  6  5  2
# 
# [[4]]
#  [1]  5  9  7  4  9  1  8  6  9  8  7  8  4 10 10
# 
# [[5]]
#  [1]  9  4 10  7  1  1  1  1  5  2  6  8 10  5  9
```


You could do that by intersecting each element in turn:


```r
intersect(intersect(intersect(intersect(l[[1]], l[[2]]), 
  l[[3]]), l[[4]]), l[[5]])
# [1] 6 9 5 8 4
```


That's hard to read because of the dagwood sandwich problem, and is equivalent to:


```r
Reduce(intersect, l)
# [1] 6 9 5 8 4
```



### Predicate functionals

A __predicate__ is a function that returns a single `TRUE` or `FALSE`, like `is.character`, `all`, or `is.NULL`.  `is.na` isn't a predicate function because it returns a vector of values. Predicate functionals make it easy to apply predicates to lists or data frames.  There are a three useful predicate functionals in base R: `Filter()`, `Find()` and `Position()`.

* `Filter`: returns a new vector containing only elements where the predicate is `TRUE`.

* `Find()`: return the first element that matches the predicate (or the last element if `right = TRUE`).

* `Position()`: return the position of the first element that matches the predicate (or the last element if `right = TRUE`).

Another useful functional makes it easy to generate a logical vector from a list (or a data frame) and a predicate:


```r
where <- function(x, f) {
  vapply(x, f, logical(1))
}
```


The following example shows how you might use these functionals with a data frame:


```r
str(Filter(is.character, iris))
# 'data.frame':	150 obs. of  0 variables
where(iris, is.character)
# Sepal.Length  Sepal.Width Petal.Length  Petal.Width      Species 
#        FALSE        FALSE        FALSE        FALSE        FALSE
str(Find(is.character, iris))
#  NULL
Position(is.character, iris)
# [1] NA
```


One function I use a lot is `compact()`:


```r
compact <- function(x) Filter(function(y) !is.null(y), y)
```


It removes all non-null elements from a list - you'll see it again in the [[fuction operators]] chapter.

### Exercises

* Use `Filter()` and `vapply()` to create a function that applies a summary statistic to every column in a data frame.

* What's the relationship between `which()` and `Position()`? 

* Re-write `compact` to eliminate the anonymous function.

* Implement `Any`, a function that takes a list and a predicate function, and returns `TRUE` if the predicate function returns `TRUE` for any of the inputs. Implement the complementary `All` function.

* Implement the `span` function from Haskell, which given a list `x` and a predicate function `f`, returns the longest sequential run of elements where the predicate is true. (Hint: you might find `rle()` helpful. Make sure to read the source and figure out how it works)

## Mathematical functionals

<!-- 
  find_funs("package:stats", fun_args, "upper")
  find_funs("package:stats", fun_args, "^f$")
-->

Functionals are very common in mathematics. The limit, the maximum, the roots (the set of points where `f(x) = 0`), and the definite integral are all functionals: given a function, they return a single number (or a vector of numbers). At first glance, these functions don't seem to fit in with the theme of eliminating loops, but if you dig deeper you'll see all of them are implemented using an algorithm that involves iteration.

In this section we'll explore some of R's built-in mathematical functionals. There are three functions that work with functions that return a single numeric value:

* `integrate`: find the area under the curve given by `f`
* `uniroot`: find where `f` hits zero
* `optimise`: find location of lowest (or highest) value of `f`

Let's explore how these are used with a simple function, `sin`:


```r
integrate(sin, 0, pi)
# 2 with absolute error < 2.2e-14
uniroot(sin, pi * c(1 / 2, 3 / 2))
# $root
# [1] 3.14
# 
# $f.root
# [1] 1.22e-16
# 
# $iter
# [1] 2
# 
# $estim.prec
# [1] 6.1e-05
optimise(sin, c(0, 2 * pi))
# $minimum
# [1] 4.71
# 
# $objective
# [1] -1
optimise(sin, c(0, pi), maximum = TRUE)
# $maximum
# [1] 1.57
# 
# $objective
# [1] 1
```


In statistics, optimisation is often used for maximum likelihood estimation. Maximum likelihood estimation (MLE) is a natural fit for functional programming because we have a well defined problem domain and a general technique to solve it. In MLE, we have two sets of parameters: the data, which is fixed for a given problem, and the parameters, which will vary as we try to find the maximum. That fits naturally with closures because we can have two layers of parameters to a closure. Closures plus optimisation gives rise to an approach to solving MLE problems like the following.

First, we create a function that computes the negative log likelihood (NLL) for a given dataset. In R, it's common to use the negative since `optimise()` defaults to findinging the minimum.


```r
poisson_nll <- function(x) {
  n <- length(x)
  function(lambda) {
    n * lambda - sum(x) * log(lambda) # + terms not involving lambda
  }
}
```


With the general NLL in hand, we create two specific NLL functions for two datasets, and use `optimise()` to find the best values, given a generous starting range.


```r
nll1 <- poisson_nll(c(41, 30, 31, 38, 29, 24, 30, 29, 31, 38)) 
nll2 <- poisson_nll(c(6, 4, 7, 3, 3, 7, 5, 2, 2, 7, 5, 4, 12, 6, 9)) 

optimise(nll1, c(0, 100))$minimum
# [1] 32.1
optimise(nll2, c(0, 100))$minimum
# [1] 5.47
```


We can verify these values are correct by using the analytic solution: in this case, it's just the mean of the data values, 32.1 and 5.45. 

Another important mathmatical functional is `optim()`. It is a generalisation of `optim()` to more than one dimension. If you're interested in how `optim()` works, you might want to explore the `Rvmmin` package, which provides a pure-R implementation of R. Interestingly `Rvmmin` is no slower than `optim()`, even though it is written in R, not C: for this problem, the bottleneck is evaluating the function multiple times, not controlling the optimisation.

### Exercises

* Implement the `arg_max` function. It should take a function, and a vector of inputs, returning the elements of the input where the function returns the highest number. For example, `arg_max(-10:5, function(x) x ^ 2)` should return -10. `arg_max(-5:5, function(x) x ^ 2)` should return `c(-5, 5)`.  Also implement the matching `arg_min`.

* Challenge: read about the [fixed point algorithm](http://mitpress.mit.edu/sicp/full-text/book/book-Z-H-12.html#%_sec_1.3). Complete the exercises using R.

## Converting loops to functionals, and when it's not possible

There are a wide class of for loops that do not naturally match of the functionals we've described so far. Sometimes it's possible to torture your code to make it work, but it's usually not a good idea: for loops are verbose and not very expressive, but all R programmers are familiar with them. It takes a while before you can identify whether or not a for loop has a related vectorised solution, or a matching functional. It's also easy to go too far: trying to convert for loops that really should stay as loops.  This section provides some more resources for learning and highlights three types of loop that you shouldn't try and convert into a functional:

* modifying in place
* recursive functions
* while loops

Stackoverflow is a good resource for learning more about converting for loops to use functionals. A couple of questions and answers that I think are particularly helpful are:

* ["Alternative to loops in R"](http://stackoverflow.com/a/14520342/16632)
* ["Speed up the loop operation in R"](http://stackoverflow.com/a/2970284/16632)

You can also look for other similar questions that have cropped up since I wrote this with a [search](http://stackoverflow.com/search?tab=votes&q=%5br%5d%20for%20loop)

### Modifying in place

If you need to modify part of an existing data frame, it's often better to use a for loop. For example, the following code sample performs a variable-by-variable transformation by matching the names of a list of functions to the names of variables in a data frame.


```r
trans <- list(
  disp = function(x) x * 0.0163871,
  am = function(x) factor(x, levels = c("auto", "manual"))
)
for(var in names(trans)) {
  mtcars[[var]] <- trans[[var]](mtcars[[var]])
}
```


We couldn't normally use `lapply()` to replace this loop directly, but it is _possible_ to replace the loop with `lapply()` by using `<<-`:


```r
lapply(names(trans), function(var) {
  mtcars[[var]] <<- trans[[var]](mtcars[[var]])
})
```


We've eliminated the for loop, but our code is longer and we've had to use an unusual language feature, `<<-`. And to understand what `mtcars[[var]] <<- ...` does, you have to understand not only how `<<-` works, but also what `x[[y]] <<- z` does behind the scenes. We've taken a simple, easily understood for loop, and turned it into something few people will understand: not a good idea!

### Recursive relationships

Another case where it's hard to convert a for loop into a functional is when the relationship is defined recursively. For example, exponential smoothing smoothes data values by taking a weighted average of the current and previous point. The `exps()` function below implements exponential smoothing with a for loop.
    

```r
exps <- function(x, alpha) {
  s <- numeric(length(x) + 1)
  for (i in seq_along(s)) {
    if (i == 1) {
      s[i] <- x[i]
    } else {
      s[i] <- alpha * x[i - 1] + (1 - alpha) * s[i - 1]
    }
  }
  s
}
x <- runif(10)
exps(x, 0.5)
#  [1] 0.392 0.392 0.226 0.249 0.173 0.499 0.339 0.520 0.367 0.356 0.228
```


We can't eliminate the for loop because none of the functionals we've seen allow the output at position `i` to depend on the input  and output at position `i - 1`. 

If we encountered this pattern a lot, we could create our own function. I've called it `lbapply()`, where `lb` is short for lookback.


```r
lbapply <- function(x, f, init = x[1], ...) {
  out <- numeric(length(x))
  out[1] <- init
  for(i in seq(2, length(x))) {
    out[i] <- f(x[i - 1], out[i - 1], ...)
  }  
  out
}

f <- function(x, out, alpha) alpha * x + (1 - alpha) * out
lbapply(x, f, alpha = 0.5)
#  [1] 0.392 0.392 0.226 0.249 0.173 0.499 0.339 0.520 0.367 0.356
```


This is only worthwhile if we need this function frequently: otherwise it just places an additional cognitive burden on the reader.

Another solution for eliminate the for loop in these cases is to [solve the recurrence relation](http://en.wikipedia.org/wiki/Recurrence_relation#Solving), removing the recursion and replacing it explicit references. This requires a new set of tools, and is mathematically challenging, but it can pay off by producing a simpler function. For exponential smoothing, it is possible to rewrite in terms of `i`:


```r
exps1 <- function(x, alpha) {
  n <- length(x)
  i <- seq_along(x) - 1
  cumsum(alpha * rev(x[-1]) * (1 - alpha) ^ i[-n]) + 
    (1 - alpha) ^ (n - 1) * x[1]
}
exps1(x, 0.5)
# [1] 0.0503 0.1365 0.1633 0.2071 0.2127 0.2256 0.2264 0.2274 0.2275
```


It's arguable whether or not that's more understandable for this case, but it may be useful for your problem. We'll see another example of a function defined recursively, the Fibonacci series, in the [[SoftwareSystems]] chapter.

### While loops

Another type of looping construct in R is the `while` loop: this keeps running code until a condition is met. `while` loops are more general than `for` loops because you can rewrite every for loop as a while loop, but you can't do the opposite.  For example, this for loop:


```r
for (i in 1:10) print(i)
```


Can be turned into this while loop:


```r
i <- 1
while(i <= 10) {
  print(i)
  i <- i + 1
}
```


Not every while loop can be turned into a for loop, because many while loops don't know in advance how many times they will be run:


```r
i <- 0
while(TRUE) {
  if (runif(1) > 0.9) break
  i <- i + 1
}
```


This is a common situation when you're writing simulations: one of the random parameters in your simulation may be how many times a process occurs.

In some cases, like above, you may be able to remove the loop by recongnising some special feature of the problem. For example, the above problem is counting how many times a Bernoulli trial with p = 0.1 is run before it is successful: this is a geometric random variable so you could replace the above code with `i <- rgeom(1, 0.1)`.  Similar to solving recurrence relations, this is extremely difficult to do in general, but you'll get big gains if you can do it for your situtation. 

## A family of functions

The following case study shows how you can use functionals to start small, with very simple functions, then build them up into more complicated and featureful tools. We'll start with a simple idea, adding two numbers together, and show how we can extend it to summing multiple numbers, computing parallel sums, cumulative sums, and sums for arrays. While we'll illustrate the ideas with addition, and you can use exactly the same ideas for multiplication, smallest and largest, and string concatenation to generate a wide family of functions, including over 20 functions provided in base R.

We'll start by defining a very simple plus function, that takes two scalar arguments:


```r
add <- function(x, y) {
  stopifnot(length(x) == 1, length(y) == 1, 
    is.numeric(x), is.numeric(y))
  x + y
}
```


(We're using R's existing addition operator here, which does much more, but the focus in this section is on how we can take very very simple functions and extend them to do more).

We really should also have some way to deal with missing values. A helper function will make this a bit easier: if `x` is missing it should return `y`, if `y` is missing it should returns `x`, and if both `x` and `y` are missing then it should returns another argument to the function: `identity`. (We'll talk a bit later about while we've called it identity). This function is probably a bit more general than what we need now, but it will come in handy when you implement other binary operators.


```r
rm_na <- function(x, y, identity) {
  if (is.na(x) && is.na(y)) {
    identity
  } else if (is.na(x)) {
    y
  } else {
    x
  }  
}
rm_na(NA, 10, 0)
# [1] 10
rm_na(10, NA, 0)
# [1] 10
rm_na(NA, NA, 0)
# [1] 0
```


That allows us to write a version of `add` that can deal with missing values if needed: (and it often is!)


```r
add <- function(x, y, na.rm = FALSE) {
  if (na.rm && (is.na(x) || is.na(y))) rm_na(x, y, 0) else x + y
}
add(10, NA)
# [1] NA
add(10, NA, na.rm = TRUE)
# [1] 10
add(NA, NA)
# [1] NA
add(NA, NA, na.rm = TRUE)
# [1] 0
```


Why did we pick an identity of `0`? Why should `add(NA, NA, na.rm = TRUE)` return 0?  Well, for every other input it returns a numeric vector of length 1, so it should do that even if both arguments are missing values. We next needs to figure out what that number should be. We can use a special property of add to work this out: add is associative, which the order of addition doesn't matter. In other words, the following two function calls should return the same value:


```r
add(add(3, NA, na.rm = TRUE), NA, na.rm = TRUE)
# [1] 3
add(3, add(NA, NA, na.rm = TRUE), na.rm = TRUE)
# [1] 3
```


That implies that `add(NA, NA, na.rm = TRUE)` must be 0.

Now we have the basics working, we can extend this function to deal with more complicated inputs. The first way we might want to extend it is add more than two numbers together. This is a simple application of `Reduce`: if the input is `c(1, 2, 3)`, then we want to compute `add(1, add(2, 3))`:


```r
r_add <- function(xs, na.rm = TRUE) {
  Reduce(function(x, y) add(x, y, na.rm = na.rm), xs)
}
r_add(c(1, 4, 10))
# [1] 15
```


This looks good, but we need to test it for a few special cases:


```r
r_add(NA, na.rm = TRUE)
# [1] NA
r_add(numeric())
# NULL
```


These are incorrect: in the first case we get a missing value even thought we've explicitly asked for them to be ignored, and in the second case we get `NULL`, instead of a length 1 numeric vector (as for every other set of inputs).

The two problems are related: if we give `Reduce()` a length one vector it doesn't have anything to reduce, so it just returns the input; if we give it a length 0 input it always returns `NULL`.  There are two ways to fix this: we can concatenate `0` to every input vector, or we can use the `init` argument to `Reduce()` (which effectively does the same thing):


```r
r_add <- function(xs, na.rm = TRUE) {
  Reduce(function(x, y) add(x, y, na.rm = na.rm), c(0, xs))
}
r_add(c(1, 4, 10))
# [1] 15
r_add(NA, na.rm = TRUE)
# [1] 0
r_add(numeric())
# [1] 0
```


(This is equivalent to `sum()`)

It would also be nice to have a vectorised version of `add` so that we can give it two vectors of numbers to add in parallel. We have two ways, using `Map()` or `vapply()`, to implement this, neither of which are perfect. `Map()` returns a list (we want a numeric vector), and while `vapply()` returns a vector, we'll need to loop over the indices.

A few test cases makes sure that it behaves as we expect.  We're a bit stricter than base R here because we don't do recyclying - you could add that if you wanted, but I find problems with recycling a common source of silent bugs.


```r
v_add <- function(x, y, na.rm = TRUE) {
  stopifnot(length(x) == length(x), is.numeric(x), is.numeric(y))
  Map(function(x, y) add(x, y, na.rm = na.rm), x, y)
}

v_add <- function(x, y, na.rm = TRUE) {
  stopifnot(length(x) == length(x), is.numeric(x), is.numeric(y))
  vapply(seq_along(x), function(i) add(x[i], y[i], na.rm = na.rm),
    numeric(1))
}
v_add(1:10, 1:10)
#  [1]  2  4  6  8 10 12 14 16 18 20
v_add(numeric(), numeric())
# numeric(0)
v_add(c(1, NA), c(1, NA))
# [1] 2 0
v_add(c(1, NA), c(1, NA), na.rm = TRUE)
# [1] 2 0
```


(This is the usual behavior of `+` in R, although we have more control over missing values.)

Another variant of adding is the cumulative sum: it's like the reductive version, but we see every step along the way to the final result. This is easy to implement with `Reduce()`'s `accumuate` argument:


```r
c_add <- function(xs, na.rm = FALSE) {
  Reduce(function(x, y) add(x, y, na.rm = na.rm), xs, 
    accumulate = TRUE)
}
c_add(1:10)
#  [1]  1  3  6 10 15 21 28 36 45 55
c_add(10:1)
#  [1] 10 19 27 34 40 45 49 52 54 55
```


(This function is equivalent to `cumsum()`)

Finally, we might want to define versions for more complicated data structures like matrices.  We could create `row` and `col` variants that sum across rows and columns respectively, or we could go the whole hog and define an array version that would sum across any arbitrary dimensions of an array.  These are easy to implement: they're a combination of `add()` and `apply()`


```r
row_sum <- function(x, na.rm = TRUE) apply(x, 1, add, na.rm = na.rm)
col_sum <- function(x, na.rm = TRUE) apply(x, 2, add, na.rm = na.rm)
arr_sum <- function(x, dim, na.rm = TRUE) apply(x, dim, add, na.rm = na.rm)
```


(These are equivalent to `rowSums()` and `colSums()`)

If every function we have created already has an existing equivalent in base R, why did we bother? There are two main reasons:

* we've created all our variants from a very simple binary operator (`add`) and a well-tested functional (`Reduce`, `Map` and `apply`), so we know all the variants will behave consistently.

* we've seen the infrastructure for addition, so we can now adapt it to other operators that might not have the full suite variants in base R.

The downside of this approach is that these implementations are unlikely to be efficient (For example, `colSums(x)` is much faster than `apply(x, 2, sum)`). However, even if they don't turn out to be fast enough, they are still a good starting point because they are less likely to have bugs; and when you create faster versions (maybe using [[Rcpp]]), you can compare results to make sure your fast versions are still correct.

### Exercises

* Implement `smaller` and `larger` functions that given two inputs return either the smaller or the larger value. Implement `na.rm = TRUE`: what should the identity be? (Hint: `smaller(x, smaller(NA, NA, na.rm = TRUE), na.rm = TRUE)` must be `x`, so `smaller(NA, NA, na.rm = TRUE)` must be bigger than any other value of x.). Use `smaller` and `larger` to implement equivalents of `min`, `max`, `pmin`, `pmax`, and new functions `row_min` and `row_max`

* Create a table that has add, multiply, smaller, larger, and, and or in the columns and binary operator, reducing variant, vectorised variant, array variants in the rows.
  
  * Fill in the cells with the names of base R functions that perform each of the roles
  
  * Compare the names and arguments of the existing R functions. How consistent are they? How could you improve them?
  
  * Complete the matrix by implementing any missing functions

* How does `paste()` fit into this structure? What is the scalar binary function that underlies `paste()`? What are the `sep` and `collapse` arguments to `paste()` equivalent to? Are there are any paste variants that don't have existing R implementations?
