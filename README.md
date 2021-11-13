# GenericML: Generic machine learning inference on heterogeneous treatment effects in randomized experiments
To cite `GenericML` in publications, please use:

Welz M., Alfons, A., Demirer, M. and Chernozhukov, V. (2021). `GenericML`: An `R` package for
generic machine learning inference on heterogeneous treatment effects in randomized experiments. *GitHub
repository*. URL: https://github.com/mwelz/GenericML.


## Summary
Highly parallelized and customizable, object-oriented  `R` implementation of [Generic Machine Learning (Chernozhukov, V., Demirer, M., Duflo, E., &amp; Fernández-Val, I., 2020)](https://arxiv.org/abs/1712.04802), using the [`mlr3`](https://github.com/mlr-org/mlr3) framework. We intend to extend this implementation to a fully-fledged `R` package for the CRAN. Please note that this implementation is still work in progress and has not yet been thoroughly tested, so we cannot yet guarantee correctness or stability. 

## Installation
To install the latest (possibly unstable) development version from GitHub, you can pull this repository and install it from the `R` command line via
```
install.packages("devtools")
devtools::install_github("mwelz/GenericML")
```
If you already have package `devtools` installed, you can skip the first line.

## Community guidelines

### Report issues and request features

If you experience any bugs or issues or if you have any suggestions for additional features, please submit an issue via the *Issues* tab of this repository. Please have a look at existing issues first to see if your problem for feature request has already been discussed.

### Contribute to the package

If you want to contribute to the package, you can fork this repository and create a pull request after implementing the desired functionality.

### Ask for help

If you need help using the package, or if you are interested in collaborations related to this project, please get in touch with the [package maintainer](https://mwelz.github.io/).


## Example
We generate `n=5000` samples that adhere to a simple linear data generating process. We emulate a randomized experiment. There is no treatment effect heterogeneity since the treatment effect is constant at value two. Hence, Generic ML should not indicate the existence of treatment effect heterogeneity.

```R

### 1. Data Generation (linear, no treatment effect heterogeneity) ----
library(GenericML)

set.seed(1)
num.obs  <- 5000
num.vars <- 5

# ATE parameter
ATE <- 2

# random treatment assignment
D <- rbinom(num.obs, 1, 0.5)

# covariates
Z <- mvtnorm::rmvnorm(num.obs, mean = rep(0, num.vars), sigma = diag(num.vars))
colnames(Z) <- paste0("z", 1:num.vars)

# counterfactual outcomes
Y0 <- as.numeric(Z %*% c(2, -3, 0, 1, 2)) + rnorm(num.obs)
Y1 <- ATE + Y0

# observed outcome
Y  <- ifelse(D == 1, Y1, Y0)


### 2. Prepare the arguments for GenericML() ----

# quantile cutoffs for the GATES grouping of the estimated CATEs
quantile_cutoffs <- c(0.2, 0.4, 0.6, 0.8) # 20%, 40%, 60%, 80% quantiles

# specify the learner of the propensity score (non-penalized logistic regression here). Propensity scores can also directly be supplied.
learner_propensity_score <- "mlr3::lrn('glmnet', lambda = 0, alpha = 1)"

# specify the considered learners of the BCA and the CATE (here: elastic net, random forest, and SVM)
learners_GenericML <- c("elastic_net", "mlr3::lrn('ranger', num.trees = 100)", "mlr3::lrn('svm')")

# specify the data that shall be used for the CLAN
# here, we use all variables of Z and uniformly distributed random noise
Z_CLAN <- cbind(Z, random = runif(num.obs))

# specify the number of splits
num_splits <- 100

# specify if a HT transformation shall be used when estimating BLP and GATES
HT <- FALSE

# A list controlling the variables that shall be used in the matrix X1 for the BLP and GATES regressions. 
X1_BLP   <- setup_X1()
X1_GATES <- setup_X1()

# consider differences between group K (most affected) with groups 1 and 2, respectively.
diff_GATES <- setup_diff(subtract_from = "most",
                         subtracted = c(1,2))
diff_CLAN  <- setup_diff(subtract_from = "most",
                         subtracted = c(1,2))

# specify the significance level
significance_level <- 0.05

# specify minimum variation of predictions before Gaussian noise with variance var(Y)/20 is added.
min_variation <- 1e-05

# specify which estimator of the error covariance matrix shall be used in BLP and GATES (standard OLS covariance matrix estimator here)
vcov_BLP   <- setup_vcov()
vcov_GATES <- setup_vcov()

# specify whether of not it should be assumed that the group variances of the most and least affected groups are equal in CLAN.
equal_variances_CLAN <- FALSE

# specify the proportion of samples that shall be selected in the auxiliary set
prop_aux <- 0.5

# specify whether or not the splits and auxiliary results of the learners shall be stored
store_splits   <- TRUE
store_learners <- TRUE

# parallelization options (currently only supported on Unix systems)
parallel  <- TRUE
num_cores <- parallel::detectCores() # maximum number
seed      <- 123456
# Note that the number of cores influences the random number stream. Thus, different choices of `num_cores` may lead to different results.



### 3. Run the GenericML() functions with these arguments ----
# runtime: ~30 seconds with R version 4.1.0 on a Dell Latitude 5300 (i5-8265U CPU @ 1.60GHz × 8, 32GB RAM), running on Ubuntu 21.10. Returns a GenericML object.
genML <- GenericML(Z = Z, D = D, Y = Y,
                   learner_propensity_score = learner_propensity_score,
                   learners_GenericML = learners_GenericML,
                   num_splits = num_splits,
                   Z_CLAN = Z_CLAN,
                   HT = HT,
                   X1_BLP = X1_BLP,
                   X1_GATES = X1_GATES,
                   vcov_BLP = vcov_BLP,
                   vcov_GATES = vcov_GATES,
                   quantile_cutoffs = quantile_cutoffs,
                   diff_GATES = diff_GATES,
                   diff_CLAN = diff_CLAN,
                   equal_variances_CLAN = equal_variances_CLAN,
                   prop_aux = prop_aux,
                   significance_level = significance_level,
                   min_variation = min_variation,
                   parallel = parallel,
                   num_cores = num_cores,
                   seed = seed,
                   store_splits = store_splits,
                   store_learners = store_learners)

### 4. Analyze the output ----
# print
genML

# the line below returns the medians of the estimated  \Lambda and \bar{\Lambda}
genML$best$overview

# Get best learner for BLP
genML$best$BLP

# Get best learner for GATES and CLAN (this is the same learner)
genML$best$GATES
genML$best$CLAN


# VEIN of BLP
get_BLP(genML, plot = FALSE)
plot(genML, type = "BLP") # plot.GenericML() method
# No indication of treatment effect heterogeneity; see beta.2 coefficient

# VEIN of GATES
get_GATES(genML, plot = FALSE)
plot(genML, type = "GATES")
# No indication of heterogeneity

# VEIN of CLAN for variable 'z1'
get_CLAN(genML, variable = "z1", plot = FALSE)
plot(genML, type = "CLAN", CLAN_variable = "z1")
# No indication of heterogeneity

```

## TODO list
Here we list desirable functionalities that we hope to implement in future releases.

- [ ] Add optional monotonization of the confidence bounds;
- [ ] Make stratified sampling an argument;
- [ ] Implement print, plot, predict, summary methods for instances of proxy_CATE, proxy_BCA, BLP, GATES, propensity_score;
- [ ] return proxy_BCA object in proxy_CATE output.

## Authors
Max Welz (welz@ese.eur.nl), Andreas Alfons (alfons@ese.eur.nl), Mert Demirer (mdemirer@mit.edu), and Victor Chernozhukov (vchern@mit.edu).
