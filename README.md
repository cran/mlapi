---
title: "Developing machine learning models with 'mlapi'"
author: "Dmitry Selivanov"
date: "2022-04-24"
output: 
  rmarkdown::html_vignette:
    keep_md: true
vignette: >
  %\VignetteIndexEntry{Developing machine learning models with 'mlapi'}
  %\VignetteEngine{knitr::rmarkdown}
  %\VignetteEncoding{UTF-8}
---

# mlapi

Idea of the `mlapi` package is to provide guideline on how to implement interfaces of the machine learning models in order to have unified consistent flow. API design is mainly borrowed from very successful python `scikit-learn` package. At the moment scope is limited to the following **base classes**:

1. `mlapiEstimation`/`mlapiEstimationOnline` - models which implements supervised learning - **regression** or **classification**
1. `mlapiTransformation`/`mlapiTransformationOnline` - models which learn **transformations** of the data. For example model can learn TF-IDF on some matrix and apply it to the other holdout matrix
1. `mlapiDecomposition`/`mlapiDecompositionOnline` - models which **decompose** input matrix into two matrices (usually low rank). A good example could be matrix factorization where input matrix $X$ decomposed into 2 matrices $P$ and $Q$ so $X \approx P Q$.

All the base classes above suggest developer to implement set of methods and expose set of members. Developer should provide realization of the class which **inherits from a corresponding base class** above.

There are several **agreements** which helps to maintain consistent workflow.

1. In opposite to the most of the R packages `mlapi` defines *models to be mutable* and internally implemented as `R6` classes.
1. Model creation is a declarative process where all the model parameters should be passed to the constructor. Model creation is separate to model fitting: `model = SomeModel$new(param_1 = 1, param_2 = 10)`.
1. Depending on the base class models should implement following methods for model training:
    * **`fit`** - `mlapiEstimation`
    * **`fit_transform`** - `mlapiTransformation`, `mlapiDecomposition`
    * **`partial_fit`** - `mlapiEstimationOnline`, `mlapiTransformationOnline`, `mlapiDecompositionOnline`
1. Depending on the base class models should implement following methods for model transformations/predictions:  
    * **`predict`** - `mlapiEstimation`, `mlapiEstimationOnline`
    * **`transform`** - `mlapiTransformation`, `mlapiTransformationOnline`, `mlapiDecomposition`, `mlapiDecompositionOnline`
1. After `mlapiDecomposition`/`mlapiDecompositionOnline` model fitting field `private$components_` should be initialized (mind undescore at the end!). It should contain **matrix** $Q$ (as per $X \approx P Q$). 
1. All the methods above should **work only with matrices** - dense or sparse. Dense matrices usually are from `base` package and sparse matrices from `Matrix` package.


This allows us to create concise pipelines which easy to train and apply to new data (details in next section):

## Example in pseudocode

### Declare models


```r
# transformer:
# scaler just divide each column by std_dev
scaler = Scaler$new()

# decomposition:
# fits truncated SVD: X = U * S * V 
# or rephrasing X = P * Q where P = U * sqrt(S); Q = sqrt(S) * V
# as a result trunc_svd$fit_transform(train) returns matrix P and learns matrix Q (stores inside model)
# when trunc_svd$transform(test) is called, model use matrix Q in order to find matrix P for `test` data
trunc_svd = SVD$new(rank = 16)

# estimator:
# fit L1/L2 regularized logistic regression
logreg = LogisticRegression(L1 = 0.1, L2 = 10)
```

### Apply pipeline to the train data

```r
train %>% 
  fit_transform(scaler) %>% 
  fit_transform(trunc_svd) %>% 
  fit(logreg)
```
Now all models are fitted.

### Apply models to the new data


```r
predictions = test %>% 
  transform(scaler) %>% 
  transform(trunc_svd) %>% 
  predict(logreg)
```

# Estimator


```r
SimpleLinearModel = R6::R6Class(
  classname = "mlapiSimpleLinearModel", 
  inherit = mlapi::mlapiEstimation, 
  public = list(
    initialize = function(tol = 1e-7) {
      private$tol = tol
      super$set_internal_matrix_formats(dense = "matrix", sparse = NULL)
    },
    fit = function(x, y, ...) {
      x = super$check_convert_input(x)
      stopifnot(is.vector(y))
      stopifnot(is.numeric(y))
      stopifnot(nrow(x) == length(y))
      
      private$n_features = ncol(x)
      private$coefficients = .lm.fit(x, y, tol = private$tol)[["coefficients"]]
    },
    predict = function(x) {
      stopifnot(ncol(x) == private$n_features)
      x %*% matrix(private$coefficients, ncol = 1)
    }
  ),
  private = list(
    tol = NULL,
    coefficients = NULL,
    n_features = NULL
  )
)
```

### Usage

```r
set.seed(1)
model = SimpleLinearModel$new()
x = matrix(sample(100 * 10, replace = TRUE), ncol = 10)
y = sample(c(0, 1), 100, replace = TRUE)
model$fit(as.data.frame(x), y)
res1 = model$predict(x)
# check pipe-compatible S3 interface
res2 = predict(x, model)
identical(res1, res2)
```

```
## [1] TRUE
```


# Decomposition


```r
TruncatedSVD = R6::R6Class(
  classname = "TruncatedSVD", 
  inherit = mlapi::mlapiDecomposition, 
  public = list(
    initialize = function(rank = 10) {
      private$rank = rank
      super$set_internal_matrix_formats(dense = "matrix", sparse = NULL)
    },
    fit_transform = function(x, ...) {
      x = super$check_convert_input(x)
      private$n_features = ncol(x)
      svd_fit = svd(x, nu = private$rank, nv = private$rank, ...)
      sing_values = svd_fit$d[seq_len(private$rank)]
      result = svd_fit$u %*% diag(x = sqrt(sing_values))
      private$components_ = t(svd_fit$v %*% diag(x = sqrt(sing_values)))
      rm(svd_fit)
      rownames(result) = rownames(x)
      colnames(private$components_) = colnames(x)
      private$fitted = TRUE
      invisible(result)
    },
    transform = function(x, ...) {
      if (private$fitted) {
        stopifnot(ncol(x) == ncol(private$components_))
        lhs = tcrossprod(private$components_)
        rhs = as.matrix(tcrossprod(private$components_, x))
        t(solve(lhs, rhs))
      }
      else
        stop("Fit the model first woth model$fit_transform()!")
    }
  ),
  private = list(
    rank = NULL, 
    n_features = NULL, 
    fitted = NULL
  )
)
```

### Usage

```r
set.seed(1)
model = TruncatedSVD$new(2)
x = matrix(sample(100 * 10, replace = TRUE), ncol = 10)
x_trunc = model$fit_transform(x)
dim(x_trunc)
```

```
## [1] 100   2
```

```r
x_trunc_2 = model$transform(x)
sum(x_trunc_2 - x_trunc)
```

```
## [1] -9.428555e-12
```

```r
# check pipe-compatible S3 interface
x_trunc_2_s3 = transform(x, model)
identical(x_trunc_2, x_trunc_2_s3)
```

```
## [1] TRUE
```
