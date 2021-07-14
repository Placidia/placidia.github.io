---
layout: post
title: Why R is unsuitable for production
date: 2021-07-13
categories: General, R
---

I have worked in R for most of my career. I know it and love it, but I have to admit that it does not scale well, nor is it the best choice for robust deployments. There are many reasons for this, but here are a few of them.

## Block safety

In most languages, what happens inside a bode block remains there, with the exception of a return value or final expression. Not so in R. Consider the following lines of code:

```r
branch <- TRUE

if (branch){
  x <- "foo"
  print("I'm true")
} else {
  y <- "bar"
  print("I'm false")
}

print(x)
print(y)
```

Upon execution, this will yield *foo* for `print(x)` and throw an error for `print(y)`. Now compare to similar code in Scala:

```scala
val branch = true
if (branch) {
    val x = "foo"
    println("I'm true")
}
else {
    val y = "bar"
    println("I'm false")
}

println(x)
println(y)
```

If I execute this in a notebook, I get **not found** errors when attempting to print either `x` or `y`. `y` was never created, and `x` vanished at the end of the block. 

### Why do we care?

In R, not only the value, but even the existence of a variable, may depend on the branch taken in a particular control sequence. And since we're doing data analysis here, every time we run our code with new data, we can expect to pass through different code branches. This, in turn, means that could end up with different variables floating around in the code, depending on the route taken through that code. These variables could interact with code further down the script in unexpected ways, creating bugs that would be difficult to squish.

### It gets worse

Most R work is done interactively in a console or notebook. Let's suppose I change the value of `branch` to `FALSE` and rerun my code cell. `x` and `y` will both print out as *foo* and *bar* respectively, with no errors. `x` is not created on this pass through the code, but it is still hanging around from the previous pass when `branch` was `TRUE`. 

## Environments

*It's not a bug; it's a feature.* One proud feature of R is its environment system. An environment is simply a collection of R objects. The objects created in an R session at the console all belong to the same environment, `.GlobalEnv`. The body of a function creates another environment, nested within `.GlobalEnv`. R packages also create environments. Function `search()` lists package environments and `.GlobalEnv`, but does not list environments created by functions. Consider this example.

```r
envir_fun <- function(){
  print(search())
  cat("\n")
  x <- "foo"
  y <- "bar"
  print(ls())
  cat("\n")
  print(ls(1))
  cat("\n")
  print(ls(2)[1:3])
}

envir_fun()
```

`envir_fun` runs the `search()` function, creates a couple of variables, and then lists the objects of the current environment and the next two environments above it. Here is the output.

```
[1] ".GlobalEnv"        "package:stats"     "package:graphics" 
[4] "package:grDevices" "package:utils"     "package:datasets" 
[7] "package:methods"   "Autoloads"         "package:base"     

[1] "x" "y"

[1] "envir_fun" "greeting"  "s2"       

[1] "acf"       "acf2AR"    "add.scope"
```

We can see the `.GlobalEnv` and the default R packages. The function environments contains `x` and `y`, as it should. The enclosing environment contains the calling function `envir_fun` and a couple of objects that I created further up the script. Then we get the first three functions from package `stats`.

An R function can work on variables defined in any of its enclosing environments.


```r 
s2 <- "bar"

greeting <- function(s1){
  output <- paste("Hello", s1, s2, sep = " ")
  red_herring <- 3
  print(output)
}

greeting("foo")

red_herring
```

This code prints out `Hello foo bar`, drawing the value of  `s2` from the calling environment. It throws a `not found` error when it attempts to print `red_herring`.

#### Why do we care?

Scala exhibits the same behaviour, but in Scala, we are less likely to lose control of `s2`. Scala allows and encourages the creation of immutable variables, so `s2` is not going to get randomly changed from its original value. And as an object oriented language, good design would suggest putting all the objects related to a particular procedure within the same class. Even if a method goes outside its function block for something, it shoudn't have to go far. 

### Dataframes and the formula language

An R dataframe also defines an environment, nested in the environment that created it. Let dataframe `df` contain vectors `y, x1, x2`. I can run a linear model of `y` against `x1` and `x2` thus:

```r
lm(y ~ x1 + x2, data = df)
```

But if `x3`, say, is a numeric vector of the same length as `y` in `.GlobalEnv`, I can do the following, with no complaint from R.

```r
lm(y ~ x1 + x2 + x3, data = df)
```

This is an open invitation to tripping over one's own work and I advise against it.

## Why R?

R is the open source implementation of `S`, as described by Becker, Chambers and Wilkes in *The New S Language*. The subtitle of their book is *A programming environment for data analysis and graphics*. So while it was always possible to run an S or R script from the command line, the primary use case was for a single statistician working interactively in a console or notebook. If you look at the code for some of the most significant packages, you can see that they typically involve a large number of short functions, to be accessed by the user as needed. When a function needs to do some heavy lifting, it typically makes a call to C or Fortran. This is a different philosophy than you find in production oriented languages.

R is technically an object oriented language, but the purpose of R classes is largely to organize data and results, not to hide or encapsulate anything. R tries to give the user access to anything one might need for the analysis. S3 classes, the most commonly used of R's class systems, contain no private members.

Two main concerns for anyone writing production code are:

* Is this code easy to understand by someone who is not the author?
* Can this code be modified without breaking anything? 

Most R code is written as part of an exploratory analysis, or intended as a "one-off" project. And while we want our code to be clear, it tpyically does not have a long shelf life. It was never inteded to.
