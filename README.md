
<!-- README.md is generated from README.Rmd. Please edit that file -->

# astgrepr

<!-- badges: start -->

[![R-CMD-check](https://github.com/etiennebacher/astgrepr/actions/workflows/R-CMD-check.yaml/badge.svg)](https://github.com/etiennebacher/astgrepr/actions/workflows/R-CMD-check.yaml)
<!-- badges: end -->

`astgrepr` provides R bindings to the
[ast-grep](https://ast-grep.github.io/) Rust crate. `ast-grep` is a tool
to parse the abstract syntax tree (AST) of some code and to perform
search and rewrite of code. This is extremely useful to build linters,
stylers, and perform a lot of code analysis.

See the example below and the [“Getting started”
vignette](https://astgrepr.etiennebacher.com/articles/astgrepr) for a
gentle introduction to `astgrepr`.

Since `astgrepr` can be used as a low-level foundation for other tools
(such as linters), the number of R dependencies is kept low:

``` r
> pak::local_deps_tree()
✔ Loading metadata database ... done
local::. 0.0.1 [new][bld]                                                  
├─checkmate 2.3.1 [new][dl] (746.54 kB)
│ └─backports 1.5.0 [new]
├─rrapply 1.2.7 [new]
└─yaml 2.3.8 [new][dl] (119.08 kB)

Key:  [new] new | [dl] download | [bld] build
```

## Installation

### Windows or macOS

``` r
install.packages(
  'astgrepr', 
  repos = c('https://etiennebacher.r-universe.dev', getOption("repos"))
)
```

### Linux

``` r
install.packages(
  'astgrepr', 
  repos = c('https://etiennebacher.r-universe.dev/bin/linux/jammy/4.3', getOption("repos"))
)
```

## Demo

``` r
library(astgrepr)

src <- "library(tidyverse)
x <- rnorm(100, mean = 2)
any(duplicated(y))
plot(x)
any(duplicated(x))
any(is.na(variable))"

root <- src |> 
  tree_new() |> 
  tree_root()

# get everything inside rnorm()
root |> 
  node_find(ast_rule(pattern = "rnorm($$$A)")) |> 
  node_get_multiple_matches("A") |> 
  node_text_all()
#> $rule_1
#> $rule_1[[1]]
#> [1] "100"
#> 
#> $rule_1[[2]]
#> [1] ","
#> 
#> $rule_1[[3]]
#> [1] "mean = 2"
```

``` r

# find occurrences of any(duplicated())
root |> 
  node_find_all(ast_rule(pattern = "any(duplicated($A))")) |> 
  node_text_all()
#> $rule_1
#> $rule_1$node_1
#> [1] "any(duplicated(y))"
#> 
#> $rule_1$node_2
#> [1] "any(duplicated(x))"
```

``` r

# find some nodes and replace them with something else
nodes_to_replace <- root |>
  node_find_all(
    ast_rule(id = "any_na", pattern = "any(is.na($VAR))"),
    ast_rule(id = "any_dup", pattern = "any(duplicated($VAR))")
  )

fixes <- nodes_to_replace |>
  node_replace_all(
    any_na = "anyNA(~~VAR~~)",
    any_dup = "anyDuplicated(~~VAR~~) > 0"
  )

# original code
cat(src)
#> library(tidyverse)
#> x <- rnorm(100, mean = 2)
#> any(duplicated(y))
#> plot(x)
#> any(duplicated(x))
#> any(is.na(variable))
```

``` r

# new code
tree_rewrite(root, fixes)
#> library(tidyverse)
#> x <- rnorm(100, mean = 2)
#> anyDuplicated(y) > 0
#> plot(x)
#> anyDuplicated(x) > 0
#> anyNA(variable)
```

## Related tools

There is some recent work linking `tree-sitter` and R. Those are not
competing with `astgrepr` but are rather a complement to it:

- [`r-lib/tree-sitter-r`](https://github.com/r-lib/tree-sitter-r):
  provide the R grammar to be used with tools built on `tree-sitter`.
  `astgrepr` relies on this grammar under the hood.
- [`DavisVaughan/r-tree-sitter`](https://github.com/DavisVaughan/r-tree-sitter):
  a companion of `r-lib/tree-sitter-r`. This gives a way to get the
  tree-sitter representation of some code directly in R. This is useful
  to learn how tree-sitter represents the R grammar, which is required
  if you want advanced use of `astgrepr`. However, it doesn’t provide a
  way to easily select specific nodes (e.g. based on patterns).
