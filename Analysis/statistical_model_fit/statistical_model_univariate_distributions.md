
# **Modelling the univariate distributions of storm event statistics**
--------------------------------------------------------------------------

*Gareth Davies, Geoscience Australia 2017*

# Introduction
------------------

This document follows on from
[statistical_model_storm_timings.md](statistical_model_storm_timings.md)
in describing our statistical analysis of storm waves at Old Bar. 

It illustrates the process of fitting probability distributions to the storm event summary statistics,
which are conditional on the time of year and ENSO.

It is essential that the code
[statistical_model_storm_timings.md](statistical_model_storm_timings.md) has
alread been run, and produced an Rdata file
*'Rimages/session_storm_timings_FALSE_0.Rdata'*. **To make sure, the code below
throws an error if the latter file does not exist.**

```r
# Check that the pre-requisites exist
if(!file.exists('Rimages/session_storm_timings_FALSE_0.Rdata')){
    stop('It appears you have not yet run the code in statistical_model_storm_timings.md. It must be run before continuing')
}
```
You might wonder why the filename ends in `_FALSE_0`. Here, `FALSE`
describes where or not we perturbed the storm summary statistics before running
the fitting code. In the `FALSE` case we didn't - we're using the raw data.
However, it can be desirable to re-run the fitting code on perturbed data, to
check the impact of data discretization on the model fit. One would usually do
many such runs, and so we include a number (`_0` in this case) to distinguish
them. So for example, if the filename ended with `_TRUE_543`, you could assume
it includes a run with the perturbed data (with 543 being a unique ID, that is
otherwise meaningless).

Supposing the above did not generate any errors, and you have R installed,
along with all the packages required to run this code, and a copy of the
*stormwavecluster* git repository, then you should be able to re-run the
analysis here by simply copy-pasting the code. Alternatively, it can be run
with the `knit` command in the *knitr* package: 

```r
library(knitr)
knit('statistical_model_univariate_distributions.Rmd')
```

The basic approach followed here is to:
* **Step 1: Load the previous session**
* **Step 2: Exploratory analysis of seasonal non-stationarity in event statistics**
* **Step 3: Model the distribution of each storm summary statistic, dependent on season (and mean annual SOI for wave direction)**

Later we will model the remaining joint dependence between these variables, and
simulate synthetic storm sequences. 

# **Step 1: Load the previous session and set some key parameters**
Here we re-load the session from the previous stage of the modelling. We also
set some parameters controlling the Monte-Carlo Markov-Chain (MCMC) computations 
further in the document. 
* The default parameter values should be appropriate for the analysis
herein. To save computational effort (for testing purposes) users might reduce
the `mcmc_chain_length`. To reduce memory usage, users can increase the
`mcmc_chain_thin` parameter. If using other datasets, it may be necessary to
increase the `mcmc_chain_length` to get convergence.


```r
previous_R_session_file = 'Rimages/session_storm_timings_FALSE_0.Rdata'
load(previous_R_session_file)

# Length of each MCMC chain. Should be 'large' e.g 10^6, except for test runs 
# We run multiple chains to enhance the likelihood of detecting non-convergence
# since anyway this is cheap in parallel. These are pooled for final estimates,
# but it is essential to manually check the convergence of the chains [e.g.
# by comparing high return period confidence intervals].
mcmc_chain_length = 1e+06 #1e+05 
# To reduce the data size, we can throw away all but a fraction of the mcmc
# chains. This has computational (memory) benefits if the MCMC samples are
# strongly autocorrelated, but no other advantages.
mcmc_chain_thin = 20 
```

# **Step 2: Exploratory analysis of seasonal non-stationarity in event statistics**
----------------------------------------------------------------------

**Here we plot the distribution of each storm statistic by month.** This
highlights the seasonal non-stationarity. Below we will take some steps to
check the statistical significance of this, and later will use copula-based
techniques to make the modelled univariate distribution of each variable
conditional on the time of year.

```r
# Get month as 1, 2, ... 12
month_num = as.numeric(format(event_statistics$time, '%m'))
par(mfrow=c(3,2))
for(i in 1:5){
    boxplot(event_statistics[,i] ~ month_num, xlab='Month', 
        ylab=names(event_statistics)[i], names=month.abb,
        col='grey')
    title(main = names(event_statistics)[i], cex.main=2)
}

rm(month_num)
```

![plot of chunk monthplot](figure/monthplot-1.png)

To model the seasonal non-stationarity illustrated above, we define a seasonal
variable periodic in time, of the form `cos(2*pi*(t - offset))` where the time
`t` is in years. The `offset` is a phase variable which can be optimised for
each storm summary statistic separately, to give the 'best' cosine seasonal
pattern matching the data. One way to do this is to find the value of `offset`
which maximises the rank-correlation between each storm variable and the seasonal
variable.

**Below we compute the offset for each storm summary statistic, and also assess
it's statistical significance using a permutation test.** The figure shows the
rank correlation between each variable and a seasonal variable, for each value
of `offset` in [-0.5, 0.5] (which represents all possible values). Note the
`offset` value with the strongest rank correlation may be interpreted as the
optimal offset (*here we choose the `offset` with largest negative rank
correlation, so many `offset`'s are close to zero*). 


```r
# Store some useful statistics
stat_store = data.frame(var = rep(NA, 5), phi=rep(NA,5), cor = rep(NA, 5), 
    p = rep(NA, 5), cor_05=rep(NA, 5))
stat_store$var = names(event_statistics)[1:5]

# Test these values of the 'offset' parameter
phi_vals = seq(-0.5, 0.5, by=0.01)
par(mfrow=c(3,2))
for(i in 1:5){

    # Compute spearman correlation for all values of phi, for variable i
    corrs = phi_vals*0
    for(j in 1:length(phi_vals)){
        corrs[j] =  cor(event_statistics[,i], 
            cos(2*pi*(event_statistics$startyear - phi_vals[j])),
            method='s', use='pairwise.complete.obs')
    }

    plot(phi_vals, corrs, xlab='Offset', ylab='Spearman Correlation', 
        main=names(event_statistics)[i], cex.main=2,
        cex.lab=1.5)
    grid()
    abline(v=0, col='orange')

    # Save the 'best' result
    stat_store$phi[i] = phi_vals[which.min(corrs)]
    stat_store$cor[i] = min(corrs)

    # Function to compute the 'best' correlation of season with
    # permuted data, which by definition has no significant correlation with
    # the season. We can use this to assess the statistical significance of the
    # observed correlation between each variable and the season.
    cor_phi_function<-function(i0=i){
        # Resample the data
        d0 = sample(event_statistics[,i0], size=length(event_statistics[,i0]), 
            replace=TRUE)
        # Correlation function
        g<-function(phi){ 
            cor(d0, cos(2*pi*(event_statistics$startyear - phi)), 
                method='s', use='pairwise.complete.obs')
        }
        # Find best 'phi'
        best_phi = optimize(g, c(-0.5, 0.5), tol=1.0e-06)$minimum

        return(g(best_phi))
    }
   
    # Let's get statistical significance 
    cor_boot = replicate(5000, cor_phi_function())

    # Because our optimizer minimises, the 'strongest' correlations
    # it finds are negative. Of course if 0.5 is added to phi this is equivalent
    # to a positive correlation. 
    
    qcb = quantile(cor_boot, 0.05, type=6)
    stat_store$cor_05[i] = qcb
    stat_store$p[i] = mean(cor_boot < min(corrs))

    polygon(rbind( c(-1, -qcb), c(-1, qcb), c(1, qcb), c(1, -qcb)),
        col='brown', density=10)
}

write.table(stat_store, file='seasonal_correlation_statistics.csv', sep="  &  ",
    quote=FALSE, row.names=FALSE)

rm(phi_vals, corrs)
```

![plot of chunk seasonphase1](figure/seasonphase1-1.png)
In the above figure, the shaded region represents a 95% interval for the best
correlation expected of 'random' data (i.e. a random sample of the original
data with an optimized offset).  Correlations outside the shaded interval are
unlikely to occur at random, and are intepreted as reflecting true seasonal
non-stationarity. 

Below we will make each storm summary statistic dependent on the seasonal
variable. For wave direction, the mean annual SOI value will also be treated.
Recall that relationships between mean annual SOI and storm wave direction
were established earlier (
[../preprocessing/extract_storm_events.md](../preprocessing/extract_storm_events.md),
[statistical_model_storm_timings.md](statistical_model_storm_timings.md) ). We
also found relationships between mean annual SOI and the rate of storms, and
MSL, which were treated in those sections (using the non-homogeneous poisson
process model, and the STL decomposition, respectively). Therefore, the latter
relationships are not treated in the section below, but they are included in
the overall model.


# **Step 3: Model the distribution of each storm summary statistic, dependent on season (and mean annual SOI for wave direction)**

In this section we model the distribution of each storm summary statistic, and
then make it conditional on the seasonal variable (and on mean annual SOI in
the case of wave direction only). 

The distributions of `hsig`, `duration` and `tideResid` are initially modelled
as extreme value mixture distributions. The distributions of `dir` and
`steepness` are initially modelled using non-parametric smoothing (based on the
log-spline method).

## Hsig

**Below we fit an extreme value mixture model to Hsig, using maximum
likelihood.** The model has a GPD upper tail, and a Gamma lower tail.

```r
# Get the exmix_fit routines in their own environment
evmix_fit = new.env()
source('../../R/evmix_fit/evmix_fit.R', local=evmix_fit, chdir=TRUE)

# Fit it
hsig_mixture_fit = evmix_fit$fit_gpd_mixture(
    data=event_statistics$hsig, 
    data_offset=as.numeric(hsig_threshold), 
    bulk='gamma')
```

```
## [1] "  evmix fit NLLH: " "530.237080056746"  
## [1] "  fit_optim NLLH: " "530.237080030477"  
## [1] "  Bulk par estimate0: " "0.842400448660163"     
## [3] "1.0204368965617"        "1.27267963923683"      
## [5] "-0.219878654410625"    
## [1] "           estimate1: " "0.842402977361388"     
## [3] "1.02042857261524"       "1.27268869445894"      
## [5] "-0.219876719484767"    
## [1] "  Difference: "        "-2.52870122496862e-06" "8.32394645855494e-06" 
## [4] "-9.05522210814524e-06" "-1.93492585820465e-06"
## [1] "PASS: checked qfun and pfun are inverse functions"
```

```r
# Make a plot
DU$qqplot3(event_statistics$hsig, hsig_mixture_fit$qfun(runif(100000)), 
    main='Hsig QQ-plot')
abline(0, 1, col='red'); grid()
```

![plot of chunk hsig_fitA](figure/hsig_fitA-1.png)

The above code leads to print-outs of the maximum likelihood parameter fits
achieved by different methods, and the differences between them (which are
only a few parts per million in this case). Because fitting extreme value
mixture models can be challenging, the code tries many different fits. 

During the fitting process, we also compute quantile and inverse quantile
functions for the fitted distribution. The code checks numerically that these
really are the inverse of each other, and will print information about whether
this was found to be true (*if not, there is a problem!*)

The quantile-quantile plot of the observed and fitted Hsig should fall close to
a straight line, if the fit worked. Poor fits are suggested by strong
deviations from the 1:1 line. While in this case the fit looks good, if the fit
is poor then further analysis is required. For example, it is possible that the
model fit did not converge, or that the statistical model is a poor choice for
the data.

Given that the above fit looks OK, **below we use Monte-Carlo-Markov-Chain
(MCMC) techniques to compute the Bayesian posterior distribution of the 4 model
parameters**. A few points about this process:
* The prior probability is uniform for each variable. Here we use
a very broad uniform distribution to represent an approximately
'non-informative' prior. The Gamma distribution parameters have uniform prior
over [0, 100 000 000]. The GPD threshold parameter prior is uniform
from zero to the 50th highest data point (to ensure that the tail
part of the model is fit using at least 50 data points). The GPD shape parameter
prior is uniform over [-1000 , 1000]. Note that for some other datasets, it
might be necessary to constrain the GPD shape parameter prior more strongly
than we do below, if it cannot be well estimated from the data (e.g. see the
literature for constraints are often imposed). Overall we are aiming to make
our priors reasonably 'non-informative', while still imposing pragmatic
constraints required to achieve a reasonable fit. 
* The routines update the object `hsig_mixture_fit`, so it contains
multiple chains, i.e. 'random walks' through the posterior parameter
distribution.
* Here we run 6 separate chains, with randomly chosen starting parameters, to
make it easier to detect non-convergence (i.e. to reduce the chance that a
single chain gets 'stuck' in part of the posterior distribution). The parameter
`mcmc_start_perturbation` defines the scale for that perturbation.
* It is possible that the randomly chosen start parameters are theoretically
impossible. In this case, the code will report that it had `Bad random start
parameters`, and will generate new ones.
* We use a burn-in of 1000 (i.e. the first 1000 entries in the chain are
discarded). This can assist with convergence.
* We make a simple diagnostic plot to check the MCMC convergence.
* The code runs in parallel, using 6 cores below. The parallel framework will
only work correctly on a shared memory linux machine.

```r
#' MCMC computations for later uncertainty characterisation

# Prevent the threshold parameter from exceeding the highest 50th data point
# Note that inside the fitting routine, Hsig was transformed to have lower
# bound of zero before fitting, since the Gamma distribution has a lower bound
# of zero. Hence we subtract hsig_threshold here
hsig_u_limit = sort(event_statistics$hsig, decreasing=TRUE)[50] - hsig_threshold

# Compute the MCMC chains in parallel
hsig_mixture_fit = evmix_fit$mcmc_gpd_mixture(
    fit_env=hsig_mixture_fit, 
    par_lower_limits=c(0, 0, 0, -1000.), 
    par_upper_limits=c(1e+08, 1.0e+08, hsig_u_limit, 1000),
    mcmc_start_perturbation=c(0.4, 0.4, 2., 0.2), 
    mcmc_length=mcmc_chain_length,
    mcmc_thin=mcmc_chain_thin,
    mcmc_burnin=1000,
    mcmc_nchains=6,
    mcmc_tune=c(1,1,1,1)*1,
    mc_cores=6,
    annual_event_rate=mean(events_per_year_truncated))

# Graphical convergence check of one of the chains. 
plot(hsig_mixture_fit$mcmc_chains[[1]])
```

![plot of chunk hsigmixtureFitBayes](figure/hsigmixtureFitBayes-1.png)

**Below, we investigate the parameter estimates for each chain.** If all the
changes have converged, the quantiles of each parameter estimate should be
essentially the same (although if the underlying posterior distribution is
unbounded, then of course the min/max will not converge, although all other
quantiles eventually will). We also look at the 1/100 year event Hsig implied
by each chain, and make a return level plot.

```r
# Look at mcmc parameter estimates in each chain
lapply(hsig_mixture_fit$mcmc_chains, f<-function(x) summary(as.matrix(x)))
```

```
## [[1]]
##       var1             var2             var3              var4        
##  Min.   :0.6466   Min.   :0.7430   Min.   :0.02852   Min.   :-0.4525  
##  1st Qu.:0.8149   1st Qu.:0.9633   1st Qu.:1.07858   1st Qu.:-0.2519  
##  Median :0.8447   Median :1.0145   Median :1.33017   Median :-0.2049  
##  Mean   :0.8450   Mean   :1.0252   Mean   :1.31840   Mean   :-0.2022  
##  3rd Qu.:0.8750   3rd Qu.:1.0727   3rd Qu.:1.59070   3rd Qu.:-0.1547  
##  Max.   :1.0372   Max.   :1.9289   Max.   :2.17545   Max.   : 0.1723  
## 
## [[2]]
##       var1             var2             var3              var4        
##  Min.   :0.6286   Min.   :0.7279   Min.   :0.03917   Min.   :-0.4436  
##  1st Qu.:0.8152   1st Qu.:0.9628   1st Qu.:1.07886   1st Qu.:-0.2524  
##  Median :0.8449   Median :1.0146   Median :1.33397   Median :-0.2057  
##  Mean   :0.8452   Mean   :1.0237   Mean   :1.32212   Mean   :-0.2025  
##  3rd Qu.:0.8749   3rd Qu.:1.0723   3rd Qu.:1.59486   3rd Qu.:-0.1546  
##  Max.   :1.0366   Max.   :1.7585   Max.   :2.17548   Max.   : 0.1799  
## 
## [[3]]
##       var1             var2             var3              var4        
##  Min.   :0.6533   Min.   :0.7190   Min.   :0.03674   Min.   :-0.4477  
##  1st Qu.:0.8151   1st Qu.:0.9635   1st Qu.:1.07511   1st Qu.:-0.2525  
##  Median :0.8450   Median :1.0140   Median :1.33417   Median :-0.2054  
##  Mean   :0.8451   Mean   :1.0242   Mean   :1.32027   Mean   :-0.2021  
##  3rd Qu.:0.8750   3rd Qu.:1.0719   3rd Qu.:1.59236   3rd Qu.:-0.1542  
##  Max.   :1.0388   Max.   :1.6847   Max.   :2.17548   Max.   : 0.1855  
## 
## [[4]]
##       var1             var2             var3               var4        
##  Min.   :0.6015   Min.   :0.7485   Min.   :0.003026   Min.   :-0.4461  
##  1st Qu.:0.8147   1st Qu.:0.9644   1st Qu.:1.067937   1st Qu.:-0.2519  
##  Median :0.8444   Median :1.0155   Median :1.327440   Median :-0.2051  
##  Mean   :0.8441   Mean   :1.0301   Mean   :1.311121   Mean   :-0.2017  
##  3rd Qu.:0.8744   3rd Qu.:1.0742   3rd Qu.:1.589451   3rd Qu.:-0.1540  
##  Max.   :1.0499   Max.   :2.3671   Max.   :2.175441   Max.   : 0.1534  
## 
## [[5]]
##       var1             var2             var3              var4        
##  Min.   :0.6282   Min.   :0.7533   Min.   :0.02086   Min.   :-0.4422  
##  1st Qu.:0.8149   1st Qu.:0.9634   1st Qu.:1.07810   1st Qu.:-0.2524  
##  Median :0.8447   Median :1.0144   Median :1.33410   Median :-0.2057  
##  Mean   :0.8449   Mean   :1.0251   Mean   :1.32227   Mean   :-0.2025  
##  3rd Qu.:0.8748   3rd Qu.:1.0725   3rd Qu.:1.59493   3rd Qu.:-0.1546  
##  Max.   :1.0365   Max.   :1.9085   Max.   :2.17535   Max.   : 0.2280  
## 
## [[6]]
##       var1             var2             var3              var4        
##  Min.   :0.6365   Min.   :0.7288   Min.   :0.02497   Min.   :-0.4535  
##  1st Qu.:0.8151   1st Qu.:0.9632   1st Qu.:1.07680   1st Qu.:-0.2524  
##  Median :0.8448   Median :1.0148   Median :1.33283   Median :-0.2051  
##  Mean   :0.8449   Mean   :1.0249   Mean   :1.32137   Mean   :-0.2023  
##  3rd Qu.:0.8747   3rd Qu.:1.0722   3rd Qu.:1.59457   3rd Qu.:-0.1547  
##  Max.   :1.0667   Max.   :2.0453   Max.   :2.17548   Max.   : 0.1718
```

```r
# Look at ari 100 estimates
lapply(hsig_mixture_fit$ari_100_chains, 
    f<-function(x) quantile(x, p=c(0.025, 0.5, 0.975)))
```

```
## [[1]]
##     2.5%      50%    97.5% 
## 7.065979 7.541583 8.845908 
## 
## [[2]]
##     2.5%      50%    97.5% 
## 7.067112 7.539645 8.861660 
## 
## [[3]]
##     2.5%      50%    97.5% 
## 7.067302 7.540992 8.870148 
## 
## [[4]]
##     2.5%      50%    97.5% 
## 7.066692 7.542467 8.896240 
## 
## [[5]]
##     2.5%      50%    97.5% 
## 7.068656 7.541864 8.859375 
## 
## [[6]]
##     2.5%      50%    97.5% 
## 7.066281 7.543819 8.857805
```

```r
# Look at model prediction of the maximum observed value
# (supposing we observed the system for the same length of time as the data covers)
lapply(hsig_mixture_fit$ari_max_data_chains, 
    f<-function(x) quantile(x, p=c(0.025, 0.5, 0.975)))
```

```
## [[1]]
##     2.5%      50%    97.5% 
## 6.814950 7.188413 8.097577 
## 
## [[2]]
##     2.5%      50%    97.5% 
## 6.815914 7.186867 8.107311 
## 
## [[3]]
##     2.5%      50%    97.5% 
## 6.816599 7.188597 8.114540 
## 
## [[4]]
##     2.5%      50%    97.5% 
## 6.817323 7.189250 8.124397 
## 
## [[5]]
##     2.5%      50%    97.5% 
## 6.817690 7.187819 8.101577 
## 
## [[6]]
##     2.5%      50%    97.5% 
## 6.814782 7.191128 8.102287
```

```r
# If the chains are well behaved, we can combine all 
summary(hsig_mixture_fit$combined_chains)
```

```
##        V1               V2               V3                 V4         
##  Min.   :0.6015   Min.   :0.7190   Min.   :0.003026   Min.   :-0.4535  
##  1st Qu.:0.8150   1st Qu.:0.9634   1st Qu.:1.076197   1st Qu.:-0.2523  
##  Median :0.8447   Median :1.0146   Median :1.332078   Median :-0.2053  
##  Mean   :0.8449   Mean   :1.0255   Mean   :1.319259   Mean   :-0.2022  
##  3rd Qu.:0.8748   3rd Qu.:1.0726   3rd Qu.:1.592707   3rd Qu.:-0.1545  
##  Max.   :1.0667   Max.   :2.3671   Max.   :2.175483   Max.   : 0.2280
```

```r
# If the chains are well behaved then we might want a merged 1/100 hsig
quantile(hsig_mixture_fit$combined_ari100, c(0.025, 0.5, 0.975))
```

```
##     2.5%      50%    97.5% 
## 7.067077 7.541721 8.864403
```

```r
# This is an alternative credible interval -- the 'highest posterior density' interval.
HPDinterval(as.mcmc(hsig_mixture_fit$combined_ari100))
```

```
##         lower    upper
## var1 6.981922 8.600684
## attr(,"Probability")
## [1] 0.95
```

```r
evmix_fit$mcmc_rl_plot(hsig_mixture_fit)
```

![plot of chunk hsigmixtureFitBayesB](figure/hsigmixtureFitBayesB-1.png)

**Here we use a different technique to compute the 1/100 AEP Hsig, as a
cross-check on the above analysis.** A simple Generalised Extreme Value model
fit to annual maxima is undertaken. While this technique is based on limited
data (i.e. only one observation per year), it is not dependent on our storm
event definition or choice of wave height threshold. In this sense it is quite
different to our peaks-over-threshold method above -- and thus serves as a
useful cross-check on the former results. 

```r
# Here we do an annual maximum analysis with a gev
# This avoids issues with event definition
annual_max_hsig = aggregate(event_statistics$hsig, 
    list(year=floor(event_statistics$startyear)), max)
# Remove the first and last years with incomplete data
keep_years = which(annual_max_hsig$year %in% 1986:2015)
library(ismev)
```

```
## Loading required package: mgcv
```

```
## Loading required package: nlme
```

```
## This is mgcv 1.8-12. For overview type 'help("mgcv-package")'.
```

```r
gev_fit_annual_max = gev.fit(annual_max_hsig[keep_years,2])
```

```
## $conv
## [1] 0
## 
## $nllh
## [1] 31.04738
## 
## $mle
## [1]  5.4969947  0.6503065 -0.2118537
## 
## $se
## [1] 0.1365196 0.1016247 0.1591920
```

```r
gev.prof(gev_fit_annual_max, m=100, xlow=6.5, xup=12, conf=0.95)
```

```
## If routine fails, try changing plotting interval
```

```r
title(main='Profile likehood confidence interval for 1/100 AEP Hsig \n using a GEV fit to annual maxima')
# Add vertical lines at the limits of the 95% interval
abline(v=c(6.97, 10.4), col='red', lty='dashed')
# Add vertical line at ML estimate
abline(v=7.4, col='orange')
```

![plot of chunk gevHsigFit](figure/gevHsigFit-1.png)

**Here we use copulas to determine a distribution for Hsig, conditional on the season**.
The computational details are wrapped up in a function that we source.
Essentially, the code:
* Finds the optimal seasonal `offset` for the chosen variable (hsig), and uses
this to create a function to compute the season statistic (which is hsig
specific) from the event time of year.
* Automatically chooses a copula family (based on AIC) to model dependence
between the chosen variable and the season variable, and fits the copula.
* Uses the copula to create new quantile and inverse quantile functions, for
which the user can pass conditional variables (i.e. to get the distribution,
given that the season variable attains a particular value).
* Test that the quantile and inverse quantile functions really are inverses of
each other (this can help catch user input errors)
* Make quantile-quantile plots of the data and model for a number of time
periods (here the first, middle and last thirds of the calendar year). The top
row shows the model with the distribution varying by season, and the bottom row
shows the model without seasonal effects. It is not particularly easy to
visually detect seasonal non-stationarities in these plots [compared, say, with
using monthly boxplots].  Their main purpose is compare the model and data
distribution at different times of year, and so detect poor model fits.
However, you might notice that the top row of plots 'hug' the 1:1 line slightly
better than corresponding plots in the bottom row in the central data values.
This reflects the modelled seasonal non-stationarities. *Note the tail behaviour
can be erratic, since the 'model' result is actually a random sample from the model.*

```r
# Get code to fit the conditional distribution
source('make_conditional_distribution.R')

# This returns an environment containing the conditional quantile and inverse
# quantile functions, among other information
hsig_fit_conditional = make_fit_conditional_on_season(
    event_statistics,
    var='hsig', 
    q_raw=hsig_mixture_fit$qfun, 
    p_raw=hsig_mixture_fit$pfun,
    startyear = 'startyear')
```

```
## [1] "Conditional p/q functions passed test: "
## [1] "  (Check plots to see if quantiles are ok)"
```

![plot of chunk fitCopulaHsig](figure/fitCopulaHsig-1.png)

```r
# What kind of copula was selected to model dependence between season and hsig?
print(hsig_fit_conditional$var_season_copula)
```

```
## Bivariate copula: Frank (par = -0.73, tau = -0.08)
```


## Duration

Here we model storm duration, using techniques virtually identical to those applied above.
As before:
* We first fit the univariate extreme value mixture distribution with maximum
likelihood; 
* Next we compute the posterior distribution of each parameter; 
* Finally we make the duration distribution conditional on the time of year, using a seasonal
variable that has been optimised to capture seasonality in the storm duration.

**Here is the extreme value mixture model maximum likelihood fit**

```r
# Do the maximum likelihood fit. Some warnings may occur during optimization as the code
# tests values of the threshold parameter = 1. This is the smallest duration value in
# the original data, and if threshold=1 then there is no data to fit the lower-tail Gamma
# model -- hence the warnings. However, for this data the best fit threshold is far above
# 1 hr, so it has no practical effect.
duration_mixture_fit = evmix_fit$fit_gpd_mixture(
    data=event_statistics$duration, 
    data_offset=0, 
    bulk='gamma')
```

```
## Warning in FUN(X[[i]], ...): initial parameter values for threshold u = 1
## are invalid

## Warning in FUN(X[[i]], ...): initial parameter values for threshold u = 1
## are invalid
```

```
## [1] "  evmix fit NLLH: " "2833.26987097606"  
## [1] "  fit_optim NLLH: " "2833.26987067979"  
## [1] "  Bulk par estimate0: " "0.787694214155467"     
## [3] "31.8814888350954"       "51.3829005439937"      
## [5] "-0.139368918503533"    
## [1] "           estimate1: " "0.787676067347033"     
## [3] "31.8814876325492"       "51.3829001974599"      
## [5] "-0.139346757730819"    
## [1] "  Difference: "        "1.81468084338166e-05"  "1.20254618707349e-06" 
## [4] "3.46533859385545e-07"  "-2.21607727144135e-05"
## [1] "PASS: checked qfun and pfun are inverse functions"
```

```r
# Make a plot
DU$qqplot3(event_statistics$duration, duration_mixture_fit$qfun(runif(100000)), 
    main='Duration QQ-plot')
abline(0, 1, col='red'); grid()
```

![plot of chunk durationMixtureML](figure/durationMixtureML-1.png)

**Here is the extreme value mixture model posterior probability computation, using MCMC**
As before, note that we run a number of MCMC chains with random starting values, and in 
the event that the random starting parameters are invalid the code will simply try new ones.

```r
#' MCMC computations for later uncertainty characterisation

# Prevent the threshold parameter from exceeding the highest 50th data point
# Unlike the case of hsig, there is no need to transform duration beforehand
duration_u_limit = sort(event_statistics$duration, decreasing=TRUE)[50]

# Compute the MCMC chains in parallel.
duration_mixture_fit = evmix_fit$mcmc_gpd_mixture(
    fit_env=duration_mixture_fit, 
    par_lower_limits=c(0, 0, 0, -1000.), 
    par_upper_limits=c(1e+08, 1.0e+08, duration_u_limit, 1000),
    mcmc_start_perturbation=c(0.4, 0.4, 2., 0.2), 
    mcmc_length=mcmc_chain_length,
    mcmc_thin=mcmc_chain_thin,
    mcmc_burnin=1000,
    mcmc_nchains=6,
    mcmc_tune=c(1,1,1,1)*1,
    mc_cores=6,
    annual_event_rate=mean(events_per_year_truncated))

# Graphical convergence check of one of the chains. 
plot(duration_mixture_fit$mcmc_chains[[1]])
```

![plot of chunk durationmixtureFitBayes](figure/durationmixtureFitBayes-1.png)

**Here we check the similarity of all the MCMC chains, and make a return-level plot for storm duration**

```r
# Look at mcmc parameter estimates in each chain
lapply(duration_mixture_fit$mcmc_chains, f<-function(x) summary(as.matrix(x)))
```

```
## [[1]]
##       var1             var2            var3             var4         
##  Min.   :0.6205   Min.   :23.92   Min.   : 2.608   Min.   :-0.32188  
##  1st Qu.:0.7612   1st Qu.:30.44   1st Qu.:40.110   1st Qu.:-0.15672  
##  Median :0.7877   Median :31.92   Median :51.125   Median :-0.10425  
##  Mean   :0.7881   Mean   :32.07   Mean   :49.167   Mean   :-0.09796  
##  3rd Qu.:0.8140   3rd Qu.:33.55   3rd Qu.:60.617   3rd Qu.:-0.04664  
##  Max.   :0.9604   Max.   :51.35   Max.   :71.000   Max.   : 0.62812  
## 
## [[2]]
##       var1             var2            var3             var4         
##  Min.   :0.6145   Min.   :24.28   Min.   : 2.916   Min.   :-0.32882  
##  1st Qu.:0.7613   1st Qu.:30.39   1st Qu.:40.334   1st Qu.:-0.15636  
##  Median :0.7877   Median :31.90   Median :51.330   Median :-0.10435  
##  Mean   :0.7883   Mean   :32.06   Mean   :49.311   Mean   :-0.09789  
##  3rd Qu.:0.8143   3rd Qu.:33.53   3rd Qu.:60.609   3rd Qu.:-0.04611  
##  Max.   :0.9563   Max.   :56.56   Max.   :70.999   Max.   : 0.41006  
## 
## [[3]]
##       var1             var2            var3             var4         
##  Min.   :0.6362   Min.   :23.43   Min.   : 3.218   Min.   :-0.32163  
##  1st Qu.:0.7616   1st Qu.:30.40   1st Qu.:40.382   1st Qu.:-0.15716  
##  Median :0.7878   Median :31.91   Median :51.209   Median :-0.10435  
##  Mean   :0.7884   Mean   :32.05   Mean   :49.255   Mean   :-0.09847  
##  3rd Qu.:0.8143   3rd Qu.:33.54   3rd Qu.:60.572   3rd Qu.:-0.04687  
##  Max.   :0.9616   Max.   :50.14   Max.   :71.000   Max.   : 0.36761  
## 
## [[4]]
##       var1             var2            var3             var4         
##  Min.   :0.6162   Min.   :23.41   Min.   : 3.897   Min.   :-0.32553  
##  1st Qu.:0.7618   1st Qu.:30.41   1st Qu.:40.424   1st Qu.:-0.15778  
##  Median :0.7878   Median :31.91   Median :51.159   Median :-0.10472  
##  Mean   :0.7884   Mean   :32.06   Mean   :49.274   Mean   :-0.09846  
##  3rd Qu.:0.8144   3rd Qu.:33.53   3rd Qu.:60.595   3rd Qu.:-0.04669  
##  Max.   :0.9668   Max.   :53.76   Max.   :71.000   Max.   : 0.42695  
## 
## [[5]]
##       var1             var2            var3            var4         
##  Min.   :0.6405   Min.   :22.99   Min.   : 2.19   Min.   :-0.32144  
##  1st Qu.:0.7615   1st Qu.:30.41   1st Qu.:40.29   1st Qu.:-0.15664  
##  Median :0.7877   Median :31.92   Median :51.22   Median :-0.10437  
##  Mean   :0.7882   Mean   :32.06   Mean   :49.28   Mean   :-0.09826  
##  3rd Qu.:0.8142   3rd Qu.:33.52   3rd Qu.:60.60   3rd Qu.:-0.04664  
##  Max.   :0.9503   Max.   :48.18   Max.   :71.00   Max.   : 0.38305  
## 
## [[6]]
##       var1             var2            var3             var4         
##  Min.   :0.6360   Min.   :23.94   Min.   : 4.216   Min.   :-0.35098  
##  1st Qu.:0.7615   1st Qu.:30.41   1st Qu.:40.187   1st Qu.:-0.15681  
##  Median :0.7877   Median :31.91   Median :51.191   Median :-0.10457  
##  Mean   :0.7881   Mean   :32.07   Mean   :49.233   Mean   :-0.09818  
##  3rd Qu.:0.8140   3rd Qu.:33.54   3rd Qu.:60.506   3rd Qu.:-0.04643  
##  Max.   :0.9568   Max.   :52.86   Max.   :71.000   Max.   : 0.36978
```

```r
# Look at ari 100 estimates
lapply(duration_mixture_fit$ari_100_chains, 
    f<-function(x) quantile(x, p=c(0.025, 0.5, 0.975)))
```

```
## [[1]]
##     2.5%      50%    97.5% 
## 150.5976 176.1429 253.9155 
## 
## [[2]]
##     2.5%      50%    97.5% 
## 150.4578 176.2841 255.0322 
## 
## [[3]]
##     2.5%      50%    97.5% 
## 150.4307 175.9921 253.7060 
## 
## [[4]]
##     2.5%      50%    97.5% 
## 150.3411 175.9763 255.6877 
## 
## [[5]]
##     2.5%      50%    97.5% 
## 150.4720 176.0949 253.5187 
## 
## [[6]]
##     2.5%      50%    97.5% 
## 150.4898 176.0385 255.9403
```

```r
# Look at model prediction of the maximum observed value
# (supposing we observed the system for the same length of time as the data covers)
lapply(duration_mixture_fit$ari_max_data_chains, 
    f<-function(x) quantile(x, p=c(0.025, 0.5, 0.975)))
```

```
## [[1]]
##     2.5%      50%    97.5% 
## 137.8663 156.2340 204.8785 
## 
## [[2]]
##     2.5%      50%    97.5% 
## 137.7779 156.2424 205.3722 
## 
## [[3]]
##     2.5%      50%    97.5% 
## 137.8023 156.1485 205.1047 
## 
## [[4]]
##     2.5%      50%    97.5% 
## 137.6786 156.0961 205.9220 
## 
## [[5]]
##     2.5%      50%    97.5% 
## 137.8279 156.1344 204.9668 
## 
## [[6]]
##     2.5%      50%    97.5% 
## 137.7901 156.1093 205.7678
```

```r
# If the chains seem ok, we can combine all 
summary(duration_mixture_fit$combined_chains)
```

```
##        V1               V2              V3              V4          
##  Min.   :0.6145   Min.   :22.99   Min.   : 2.19   Min.   :-0.35098  
##  1st Qu.:0.7615   1st Qu.:30.41   1st Qu.:40.29   1st Qu.:-0.15688  
##  Median :0.7877   Median :31.91   Median :51.21   Median :-0.10444  
##  Mean   :0.7883   Mean   :32.06   Mean   :49.25   Mean   :-0.09820  
##  3rd Qu.:0.8142   3rd Qu.:33.54   3rd Qu.:60.58   3rd Qu.:-0.04656  
##  Max.   :0.9668   Max.   :56.56   Max.   :71.00   Max.   : 0.62812
```

```r
# If the chains are well behaved then we might want a merged 1/100 hsig
quantile(duration_mixture_fit$combined_ari100, c(0.025, 0.5, 0.975))
```

```
##     2.5%      50%    97.5% 
## 150.4614 176.0909 254.6940
```

```r
HPDinterval(as.mcmc(duration_mixture_fit$combined_ari100))
```

```
##         lower    upper
## var1 145.8444 237.4711
## attr(,"Probability")
## [1] 0.95
```

```r
# Return level plot
evmix_fit$mcmc_rl_plot(duration_mixture_fit)
```

![plot of chunk durationMCMCcheck](figure/durationMCMCcheck-1.png)

**Finally we make the duration fit conditional on the time of year, using a seasonal variable.**

```r
# This returns an environment containing the conditional quantile and inverse
# quantile functions, among other information
duration_fit_conditional = make_fit_conditional_on_season(
    event_statistics,
    var='duration', 
    q_raw=duration_mixture_fit$qfun, 
    p_raw=duration_mixture_fit$pfun,
    startyear = 'startyear')
```

```
## [1] "Conditional p/q functions passed test: "
## [1] "  (Check plots to see if quantiles are ok)"
```

![plot of chunk fitCopulaDuration](figure/fitCopulaDuration-1.png)

```r
# What kind of copula was selected to model dependence between season and duration?
print(duration_fit_conditional$var_season_copula)
```

```
## Bivariate copula: Gaussian (par = -0.15, tau = -0.09)
```

## Tidal residual

Here we generally follow the steps implemented above for Hsig and duration. An
important change is that we fit an extreme value mixture model with a normal
lower tail (instead of a Gamma lower tail). This is done because unlike storm
Hsig and duration, there is no natural lower limit on the tidal residual [e.g.
it can even be negative on occasion].


```r
# Manually remove missing (NA) data before fitting
tideResid_mixture_fit = evmix_fit$fit_gpd_mixture(
    data=na.omit(event_statistics$tideResid),  
    bulk='normal')
```

```
## [1] "  evmix fit NLLH: " "-443.491563137327" 
## [1] "  fit_optim NLLH: " "-443.491563173242" 
## [1] "  Bulk par estimate0: " "0.114681095472333"     
## [3] "0.1135725221079"        "0.185571006249087"     
## [5] "-0.125692215113564"    
## [1] "           estimate1: " "0.114681127821652"     
## [3] "0.113572500789905"      "0.185570939525485"     
## [5] "-0.125692214524754"    
## [1] "  Difference: "        "-3.23493184045676e-08" "2.13179948138631e-08" 
## [4] "6.67236016438366e-08"  "-5.88809556667513e-10"
## [1] "PASS: checked qfun and pfun are inverse functions"
```

```r
# Make a plot
DU$qqplot3(na.omit(event_statistics$tideResid), 
    tideResid_mixture_fit$qfun(runif(100000)))
abline(0, 1, col='red')
grid()
```

![plot of chunk tideResidExtremeValueMixture](figure/tideResidExtremeValueMixture-1.png)

Below is the MCMC computation of the posterior probability distribution.
As before, bad random starting parameters are rejected, with a warning.

```r
#' MCMC computations for later uncertainty characterisation
min_tr = min(event_statistics$tideResid, na.rm=TRUE)
tideResid_u_limit = sort(event_statistics$tideResid, decreasing=TRUE)[50]

tideResid_mixture_fit = evmix_fit$mcmc_gpd_mixture(
    fit_env=tideResid_mixture_fit, 
    par_lower_limits=c(min_tr, 0, min_tr, -1000), 
    par_upper_limits=c(1e+08, 1e+08, tideResid_u_limit, 1000),
    mcmc_start_perturbation=c(0.2, 0.2, 0.2, 0.3), 
    mcmc_length=mcmc_chain_length,
    mcmc_thin=mcmc_chain_thin,
    mcmc_burnin=1000,
    mcmc_nchains=6,
    mcmc_tune=c(1,1,1,1)*1.,
    mc_cores=6,
    annual_event_rate=mean(events_per_year_truncated))

# Graphical convergence check
plot(tideResid_mixture_fit$mcmc_chains[[1]])
```

![plot of chunk tideResidMCMC](figure/tideResidMCMC-1.png)

```r
# Clean up
rm(min_tr)
```

**Here we further investigate the behaviour of the MCMC chains for the tidal residual fit,
and make a return-level plot**

```r
# Look at mcmc parameter estimates in each chain
lapply(tideResid_mixture_fit$mcmc_chains, f<-function(x) summary(as.matrix(x)))
```

```
## [[1]]
##       var1              var2              var3             var4          
##  Min.   :0.09368   Min.   :0.09983   Min.   :0.1166   Min.   :-0.251995  
##  1st Qu.:0.11263   1st Qu.:0.11311   1st Qu.:0.1994   1st Qu.:-0.109163  
##  Median :0.11583   Median :0.11572   Median :0.2289   Median :-0.060319  
##  Mean   :0.11583   Mean   :0.11574   Mean   :0.2300   Mean   :-0.043888  
##  3rd Qu.:0.11899   3rd Qu.:0.11835   3rd Qu.:0.2633   3rd Qu.: 0.003937  
##  Max.   :0.13623   Max.   :0.13079   Max.   :0.2929   Max.   : 0.573912  
## 
## [[2]]
##       var1              var2              var3             var4          
##  Min.   :0.09571   Min.   :0.09983   Min.   :0.1074   Min.   :-0.244458  
##  1st Qu.:0.11264   1st Qu.:0.11310   1st Qu.:0.1993   1st Qu.:-0.108707  
##  Median :0.11586   Median :0.11573   Median :0.2293   Median :-0.059997  
##  Mean   :0.11587   Mean   :0.11574   Mean   :0.2301   Mean   :-0.043581  
##  3rd Qu.:0.11912   3rd Qu.:0.11838   3rd Qu.:0.2636   3rd Qu.: 0.003095  
##  Max.   :0.13516   Max.   :0.13279   Max.   :0.2929   Max.   : 0.643393  
## 
## [[3]]
##       var1              var2             var3             var4          
##  Min.   :0.09507   Min.   :0.1006   Min.   :0.1183   Min.   :-0.239670  
##  1st Qu.:0.11258   1st Qu.:0.1131   1st Qu.:0.1988   1st Qu.:-0.109961  
##  Median :0.11581   Median :0.1157   Median :0.2284   Median :-0.061144  
##  Mean   :0.11582   Mean   :0.1157   Mean   :0.2295   Mean   :-0.044826  
##  3rd Qu.:0.11907   3rd Qu.:0.1183   3rd Qu.:0.2628   3rd Qu.: 0.002408  
##  Max.   :0.13680   Max.   :0.1309   Max.   :0.2929   Max.   : 0.549930  
## 
## [[4]]
##       var1              var2              var3             var4          
##  Min.   :0.09608   Min.   :0.09954   Min.   :0.1145   Min.   :-0.244409  
##  1st Qu.:0.11254   1st Qu.:0.11312   1st Qu.:0.1994   1st Qu.:-0.109371  
##  Median :0.11580   Median :0.11574   Median :0.2294   Median :-0.060195  
##  Mean   :0.11581   Mean   :0.11575   Mean   :0.2301   Mean   :-0.044062  
##  3rd Qu.:0.11905   3rd Qu.:0.11838   3rd Qu.:0.2633   3rd Qu.: 0.002347  
##  Max.   :0.13675   Max.   :0.13324   Max.   :0.2929   Max.   : 0.720116  
## 
## [[5]]
##       var1              var2              var3             var4          
##  Min.   :0.09662   Min.   :0.09999   Min.   :0.1151   Min.   :-0.233873  
##  1st Qu.:0.11259   1st Qu.:0.11307   1st Qu.:0.1986   1st Qu.:-0.109852  
##  Median :0.11582   Median :0.11571   Median :0.2282   Median :-0.061021  
##  Mean   :0.11583   Mean   :0.11571   Mean   :0.2294   Mean   :-0.044785  
##  3rd Qu.:0.11909   3rd Qu.:0.11833   3rd Qu.:0.2626   3rd Qu.: 0.002891  
##  Max.   :0.13626   Max.   :0.13256   Max.   :0.2929   Max.   : 0.739774  
## 
## [[6]]
##       var1              var2              var3             var4          
##  Min.   :0.09505   Min.   :0.09988   Min.   :0.1198   Min.   :-0.233305  
##  1st Qu.:0.11259   1st Qu.:0.11310   1st Qu.:0.1988   1st Qu.:-0.109896  
##  Median :0.11582   Median :0.11571   Median :0.2285   Median :-0.061764  
##  Mean   :0.11583   Mean   :0.11572   Mean   :0.2295   Mean   :-0.045082  
##  3rd Qu.:0.11903   3rd Qu.:0.11835   3rd Qu.:0.2627   3rd Qu.: 0.002322  
##  Max.   :0.13450   Max.   :0.13437   Max.   :0.2929   Max.   : 0.722927
```

```r
# Look at ari 100 estimates
lapply(tideResid_mixture_fit$ari_100_chains, 
    f<-function(x) quantile(x, p=c(0.025, 0.5, 0.975)))
```

```
## [[1]]
##      2.5%       50%     97.5% 
## 0.5366436 0.6140085 0.8719167 
## 
## [[2]]
##      2.5%       50%     97.5% 
## 0.5367904 0.6138660 0.8776888 
## 
## [[3]]
##      2.5%       50%     97.5% 
## 0.5366700 0.6132702 0.8735626 
## 
## [[4]]
##      2.5%       50%     97.5% 
## 0.5361377 0.6135575 0.8735141 
## 
## [[5]]
##      2.5%       50%     97.5% 
## 0.5368935 0.6135468 0.8704017 
## 
## [[6]]
##      2.5%       50%     97.5% 
## 0.5363377 0.6137062 0.8680282
```

```r
# Look at model prediction of the maximum observed value
# (supposing we observed the system for the same length of time as the data covers)
lapply(tideResid_mixture_fit$ari_max_data_chains, 
    f<-function(x) quantile(x, p=c(0.025, 0.5, 0.975)))
```

```
## [[1]]
##      2.5%       50%     97.5% 
## 0.4873812 0.5429270 0.6836023 
## 
## [[2]]
##      2.5%       50%     97.5% 
## 0.4875296 0.5426287 0.6866169 
## 
## [[3]]
##      2.5%       50%     97.5% 
## 0.4876175 0.5425375 0.6836983 
## 
## [[4]]
##      2.5%       50%     97.5% 
## 0.4871691 0.5425232 0.6844824 
## 
## [[5]]
##      2.5%       50%     97.5% 
## 0.4876506 0.5427917 0.6820844 
## 
## [[6]]
##      2.5%       50%     97.5% 
## 0.4873321 0.5428597 0.6797334
```

```r
# If the chains seem ok, we can combine all 
summary(tideResid_mixture_fit$combined_chains)
```

```
##        V1                V2                V3               V4           
##  Min.   :0.09368   Min.   :0.09954   Min.   :0.1074   Min.   :-0.251995  
##  1st Qu.:0.11259   1st Qu.:0.11310   1st Qu.:0.1990   1st Qu.:-0.109468  
##  Median :0.11583   Median :0.11572   Median :0.2288   Median :-0.060747  
##  Mean   :0.11583   Mean   :0.11573   Mean   :0.2297   Mean   :-0.044371  
##  3rd Qu.:0.11906   3rd Qu.:0.11835   3rd Qu.:0.2630   3rd Qu.: 0.002825  
##  Max.   :0.13680   Max.   :0.13437   Max.   :0.2929   Max.   : 0.739774
```

```r
# If the chains are well behaved then we might want a merged 1/100 hsig
quantile(tideResid_mixture_fit$combined_ari100, c(0.025, 0.5, 0.975))
```

```
##      2.5%       50%     97.5% 
## 0.5365757 0.6136366 0.8725682
```

```r
HPDinterval(as.mcmc(tideResid_mixture_fit$combined_ari100))
```

```
##          lower     upper
## var1 0.5193528 0.8061485
## attr(,"Probability")
## [1] 0.95
```

```r
# Return level plot
evmix_fit$mcmc_rl_plot(tideResid_mixture_fit)
```

![plot of chunk tideResidMCMCcheck](figure/tideResidMCMCcheck-1.png)

Below we make the tidal residual distribution conditional on the time of year,
via a seasonal variable with phase optimized to model tidal residual seasonality.

```r
# This returns an environment containing the conditional quantile and inverse
# quantile functions, among other information
tideResid_fit_conditional = make_fit_conditional_on_season(
    event_statistics,
    var='tideResid', 
    q_raw=tideResid_mixture_fit$qfun, 
    p_raw=tideResid_mixture_fit$pfun,
    startyear = 'startyear')
```

```
## [1] "Conditional p/q functions passed test: "
## [1] "  (Check plots to see if quantiles are ok)"
```

![plot of chunk tideResidDependence](figure/tideResidDependence-1.png)

```r
# What kind of copula was selected to model dependence between season and
# tidal residual?
print(tideResid_fit_conditional$var_season_copula)
```

```
## Bivariate copula: Gaussian (par = -0.16, tau = -0.1)
```


## **Moving On**
The next part of this tutorial begins at XXXXX.