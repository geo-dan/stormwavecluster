
# **Statistical modelling of the storm event dataset**
-------------------------------------------------------------

*Gareth Davies, Geoscience Australia 2017*

# Introduction
------------------

This document follows on from
[../preprocessing/extract_storm_events.md](../preprocessing/extract_storm_events.md)
in describing our statistical analysis of storm waves at Old Bar. 

It illustrates the process of fitting the statistical model to the data 

It is essential that the scripts in [../preprocessing](../preprocessing) have
alread been run, and produced an
RDS file *'../preprocessing/Derived_data/event_statistics.RDS'*. **To make sure, the
code below throws an error if the latter file does not exist.**

```r
# Check that the pre-requisites exist
if(!file.exists('../preprocessing/Derived_data/event_statistics.RDS')){
    stop('It appears you have not yet run all codes in ../preprocessing. They must be run before continuing')
}
```

Supposing the above did not generate any errors, and you have R installed,
along with all the packages required to run this code, and a copy of the
*stormwavecluster* git repository, then you should be able to re-run the
analysis here by simply copy-pasting the code. Alternatively, it can be run
with the `knit` command in the *knitr* package: 

```r
library(knitr)
knit('statistical_model.Rmd')
```

The basic approach followed here is to:
* **Step 1: Get the preprocessed data and address rounding artefacts**
* **Step 2: Compute the wave steepness**
* **Step 3: Exploration of the annual number of storms**
* **Step 4: Modelling the storm event timings as a non-homogeneous Poisson process**

Later we will use the statistical model to simulate synthetic storm event
time-series.

# **Step 1: Get the preprocessed data and address rounding artefacts**
----------------------------------------------------------------------


**Below we read the data from earlier steps of the analysis**

```r
# Get the data_utilities (in their own environment to keep the namespace clean)
DU = new.env()
source('../preprocessing/data_utilities.R', local=DU) 

# Read data saved by processing scripts
event_statistics_list = readRDS('../preprocessing/Derived_data/event_statistics.RDS')

# Extract variables that we need from previous analysis
for(varname in names(event_statistics_list)){
    assign(varname, event_statistics_list[[varname]])
}

# Clean up
rm(event_statistics_list)

# Definitions controlling the synthetic series creation
nyears_synthetic_series = 1e+06 #1e+03

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

# Useful number to convert from years to hours (ignoring details of leap-years)
year2hours = 365.25*24

# Optionally remove ties in the event statistics by jittering
break_ties_with_jitter = FALSE

# Look at the variables we have
ls()
```

```
##  [1] "break_ties_with_jitter"           "CI_annual_fun"                   
##  [3] "data_duration_years"              "DU"                              
##  [5] "duration_gap_hours"               "duration_offset_hours"           
##  [7] "duration_threshold_hours"         "event_statistics"                
##  [9] "hsig_threshold"                   "mcmc_chain_length"               
## [11] "mcmc_chain_thin"                  "nyears_synthetic_series"         
## [13] "obs_start_time_strptime"          "smooth_tideResid_fun_stl_monthly"
## [15] "soi_SL_lm"                        "varname"                         
## [17] "year2hours"
```

If our event statistics are subject to rounding (introducing 'ties' or repeated
values into the data), then it is possible for some statistical methods used
here to perform badly (since they assume continuous data, which has probability
zero of ties).

**Below we optionally perturb the `event_statistics` to remove ties**. The perturbation
size is related to the resolution of the data, which is 1 cm for Hsig, 1 hour for
duration, and 1 degree for direction. For tp1 (which has the most ties, and only
40 unique values), the bins are irregularly spaced without an obvious pattern.
The median distance between unique tp1 values after sorting is 0.25, with a maximum of
1.06, and a minimum of 0.01. Therefore, below a uniform perturbation of plus/minus 0.1
second is applied.


```r
# Make a function which will return a jittered version of the original
# event_statistics
make_jitter_event_statistics<-function(event_statistics_orig){
    # Save original data
    event_statistics_orig = event_statistics_orig

    # Function that will jitter the original event_statistics
    jitter_event_statistics<-function(
        jitter_vars = c('hsig', 'duration', 'dir', 'tp1'),
        # Jitter amounts = half of the bin size [see comments above regarding TP1]
        jitter_amounts = c(0.005, 0.5, 0.5, 0.1)
        ){

        event_statistics = event_statistics_orig

        # Jitter
        for(i in 1:length(jitter_vars)){ 
            event_statistics[[jitter_vars[i]]] = 
                jitter(event_statistics[[jitter_vars[i]]], 
                    amount = jitter_amounts[i])
        }

        # But hsig must be above the threshold
        kk = which(event_statistics$hsig <= hsig_threshold)
        if(length(kk) > 0){
            event_statistics$hsig[kk] = event_statistics_orig$hsig[kk]
        }

        return(event_statistics)
    }
    return(jitter_event_statistics)
}

# Function that will return a jitter of the original event_statistics
event_statistics_orig = event_statistics
jitter_event_statistics = make_jitter_event_statistics(event_statistics_orig)

if(break_ties_with_jitter){
    # Jitter the event statistics
    event_statistics = jitter_event_statistics()
    summary(event_statistics)
}
```



# **Step 2: Compute the wave steepness**
----------------------------------------

In our analysis, we prefer to model the wave steepness (a function of the wave
height and period) instead of working directly with wave period tp1. Below
the wave steepness is computed using the Airy wave dispersion relation, assuming
the water depth is 80m (which is appropriate for the MHL wave buoys).


```r
wavedisp = new.env()
source('../../R/wave_dispersion/wave_dispersion_relation.R', local=wavedisp)
buoy_depth = 80 # Depth in m. For Crowdy head the results are not very sensitive to this
wavelengths = wavedisp$airy_wavelength(period=event_statistics$tp1, h=buoy_depth)
event_statistics$steepness = event_statistics$hsig/wavelengths
rm(wavelengths)
summary(event_statistics$steepness)
```

```
##     Min.  1st Qu.   Median     Mean  3rd Qu.     Max. 
## 0.008328 0.016110 0.019850 0.020770 0.024570 0.051790
```

```r
hist(event_statistics$steepness, main='Histogram of computed wave steepness values')
```

![plot of chunk computeWaveSteepness](figure/computeWaveSteepness-1.png)

# **Step 3: Exploration of the annual number of storms, the time between storms, and seasonal patterns**
--------------------------------------------------------------------------------------------------------

**Here we plot the number of events each year in the data.** Notice
how there are few events in the first and last year. This reflects
that our data is incomplete during those years.

```r
events_per_year = table(format(event_statistics$time, '%Y'))
# Hack into numeric type
year = as.numeric(names(events_per_year))
events_per_year = as.numeric(events_per_year)

# Plot
plot(year, as.numeric(events_per_year), t='h', lwd=3, lend=1, 
    main='Number of storm events per year',
    xlab='Year',
    ylab='Events per year', 
    ylim=c(0, max(events_per_year)))
```

![plot of chunk temporal_spacing1](figure/temporal_spacing1-1.png)

```r
# Clean up
rm(year)
```

Before considering detailed modelling of the time-series, we informally check
whether the annual counts behave like a Poisson distribution. This would
imply the annual count data has a mean that is equal to its variance (though
because of finite sampling, this is not expected to hold exactly). We check
this by simulating a large number of samples from a Poisson distribution with
the same mean as our data, and computing their variance. The first and last years
of the data are removed to avoid the artefacts mentioned above.

If the data were truely Poisson, then we would expect the data variance to fall
well within the samples from the simulation -- which it does. **The code below implements
this check.**

```r
l = length(events_per_year)
events_per_year_truncated = events_per_year[-c(1,l)]

# For Poisson, mean = variance (within sampling variability)
sample_mean = mean(events_per_year_truncated)
sample_mean
```

```
## [1] 22.36667
```

```r
sample_var = var(events_per_year_truncated)
sample_var
```

```
## [1] 16.37816
```

```r
# Simulate
n = length(events_per_year_truncated)
nsim = 10000
simulated_variance = replicate(nsim, var( rpois( n, lambda=sample_mean)))
empirical_distribution = ecdf(simulated_variance)

# What fraction of the empirical samples have a variance less than our data sample?
empirical_distribution(sample_var)
```

```
## [1] 0.1544
```

```r
hist(simulated_variance, breaks=60, freq=FALSE,
    main=paste0('Distribution of the sample variance for count data \n',
                'from a Poisson distribution (mean and sample size = data) \n',
                ' (Red line is our data variance)'))
abline(v=sample_var, col='red')
```

![plot of chunk checkpoisson1](figure/checkpoisson1-1.png)

```r
# Clean up
rm(l, n, nsim, simulated_variance, sample_var, empirical_distribution)
```

The above graphical and statistical checks do not suggests any particularly
strong deviation from the Poisson model *for the annual count data*. **Below, we
examine the data on shorter time-scales by plotting the distribution of times
between events, and the number of events each season.**

```r
num_events = length(event_statistics$startyear)
time_between_events = event_statistics$startyear[2:num_events] - 
                      event_statistics$endyear[1:(num_events-1)]

par(mfrow=c(2,1))
hist(time_between_events, freq=FALSE, xlab='Time between events (years)', 
    breaks=40, main='Histogram of time between events', cex.main=1.5)

# Add an exponential decay curve with rate based on the mean (corresponding to
# ideal Poisson Process)
xs = seq(0,1,len=100)
points(xs, exp(-xs/mean(time_between_events))/mean(time_between_events), 
    t='l', col=2)
grid()

# Compute the fraction of events which occur in each month
events_per_month = aggregate(
    rep(1/num_events, num_events), 
    list(month=format(event_statistics$time, '%m')), 
    FUN=sum)

# Get month labels as Jan, Feb, Mar, ...
month_label = month.abb[as.numeric(events_per_month$month)]
barplot(events_per_month$x, names.arg = month_label, 
        main='Fraction of events occuring in each calendar month',
        cex.main=1.5)
grid()
```

![plot of chunk timebetween_seasonal](figure/timebetween_seasonal-1.png)

```r
# Clean up
rm(time_between_events, xs, events_per_month, month_label, num_events)
```


# **Step 4: Modelling the storm event timings as a non-homogeneous Poisson process**
------------------------------------------------------------------------------------

Below we use the nhpoisp.R script to fit various non-homogeneous Poisson process models
to the storm event timings. The code below is somewhat complex, since it automates the fit
of a range of different models. If you are trying to learn to fit these models, it is strongly
suggested you consult the tests and introductory illustrations contained in the folder
[../../R/nhpoisp/](../../R/nhpoisp/).

```r
nhp = new.env()
source('../../R/nhpoisp/nhpoisp.R', local=nhp)

#
# Prepare the data for modelling
#

# Get event start time in years
event_time = event_statistics$startyear
# Get event duration in years 
# NOTE: Because we demand a 'gap' between events, it makes sense to add 
#       (duration_gap_hours - duration_offset_hours) to the
#       event duration, since no other event is possible in this time.
event_duration_years = (event_statistics$duration + duration_gap_hours - 
    duration_offset_hours)/year2hours 
obs_start_time = DU$time_to_year(obs_start_time_strptime)


# We cannot use all the data for these fits which include SOI terms,
# because we don't have annual SOI for 2016+
bulk_fit_indices = which(event_time < 2016) 

annual_rate_equations = list(
    # The most basic model.
    constant=list(eqn = 'theta[1]', npar = 1, start_par = 30, par_scale=1),

    # A model where soi matters. Because we only have annual soi until 2015,
    # we make the function loop over values from 1985 - 2015 inclusive. 
    # No values are 'looped' for the actual model fit so the parameters
    # are unaffected, however, for plotting the loop helps, since we have
    # to simulate many values.
    soi = list(
        eqn='theta[1] + theta[2]*CI_annual_fun$soi(floor((t - 1985)%%31 + 1985))',
        npar=2, start_par = c(30, 0.1), par_scale=c(1,1/10))
    )


seasonal_rate_equations = list(
    # The most basic model,
    constant=list(eqn='0', npar=0, start_par=c(), par_scale=c()),

    # Simple sinusoid,
    single_freq = list(
        eqn='theta[n_annual_par + 1]*sin(2*pi*(t - theta[n_annual_par + 2]))', 
        npar=2, start_par=c(5, 0.1), par_scale = c(1, 1/10)),
        
    # Double sinusoid
    double_freq = list(eqn=paste0(
        'theta[n_annual_par + 1]*sin(2*pi*(t - theta[n_annual_par + 2])) + ',
        'theta[n_annual_par + 3]*sin(4*pi*(t - theta[n_annual_par + 4]))'),
        npar=4, start_par=c(5, 0.1, 1, 0.1), par_scale=c(1, 1/10, 1, 1/10)),

    # Sawtooth
    sawtooth = list(
        eqn='theta[n_annual_par + 1]*abs(2/pi*asin(cos(pi*(t-theta[n_annual_par+2]))))',
        npar=2, start_par=c(5, 0.5), par_scale=c(1, 1/10))
    )

cluster_rate_equations = list(
    # The most basic model
    constant=list(eqn="0", npar=0, start_par=c(), par_scale=c()),

    # A clustering model
    # Note the cluster time-scale must be positive to keep it sensible
    cluster = list(
        eqn=paste0('theta[n_annual_par + n_seasonal_par + 1]*',
            'exp((tlast - t)*abs(theta[n_annual_par + n_seasonal_par + 2]))'),
        npar=2, start_par=c(1, 100), par_scale=c(1, 100))
)

annual_rate_names = names(annual_rate_equations)
seasonal_rate_names = names(seasonal_rate_equations)
cluster_rate_names = names(cluster_rate_equations)

counter = 0
exhaustive_lambda_model_fits = list()

# Options to control the loop over all models
print_info = TRUE 
fit_model = TRUE
use_previous_fit_for_cluster_pars = FALSE

# Loop over all rate equations (representing all combinations of
# annual/seasona/cluster models)
for(ar_name in annual_rate_names){
    for(sr_name in seasonal_rate_names){
        for(cr_name in cluster_rate_names){

            counter = counter + 1
            # Make starting parameters
            if( (cr_name == 'constant') | 
                (!use_previous_fit_for_cluster_pars) | 
                !fit_model ){
                start_par = c(annual_rate_equations[[ar_name]]$start_par, 
                              seasonal_rate_equations[[sr_name]]$start_par,
                              cluster_rate_equations[[cr_name]]$start_par)
            }else{
                # Better to use the parameters from before
                start_par = c(local_fit$par, cluster_rate_equations[[cr_name]]$start_par)
            }

            # Make scaling parameters
            par_scale = c(annual_rate_equations[[ar_name]]$par_scale, 
                          seasonal_rate_equations[[sr_name]]$par_scale,
                          cluster_rate_equations[[cr_name]]$par_scale)

            # Make preliminary equation
            rate_equation = paste0(
                annual_rate_equations[[ar_name]]$eqn, '+', 
                seasonal_rate_equations[[sr_name]]$eqn, '+', 
                cluster_rate_equations[[cr_name]]$eqn)
            
            # Sub in the required parameters 
            rate_equation = gsub('n_annual_par', annual_rate_equations[[ar_name]]$npar, 
                rate_equation)
            rate_equation = gsub('n_seasonal_par', seasonal_rate_equations[[sr_name]]$npar, 
                rate_equation)
            
            if(print_info){
                print('')
                print(ar_name)
                print(sr_name)
                print(cr_name)
                print(rate_equation)
                print(start_par)
                print(par_scale)
            }

            if(fit_model){

                # For all single parameter models, just use BFGS. For others,
                # do two Nelder-Mead fits, followed by BFGS
                if(length(start_par) == 1){
                    optim_method_sequence = 'BFGS'
                }else{
                    optim_method_sequence = c('Nelder-Mead', 'Nelder-Mead', 'BFGS')
                }
                
                local_fit =  nhp$fit_nhpoisp(event_time[bulk_fit_indices],
                    rate_equation=rate_equation,
                    minimum_rate=0.0,
                    initial_theta=start_par,
                    x0 = obs_start_time,
                    event_durations = event_duration_years[bulk_fit_indices],
                    number_of_passes = 3,
                    optim_method=optim_method_sequence,
                    enforce_nonnegative_theta=FALSE,
                    optim_control=par_scale,
                    use_optim2=FALSE)

                exhaustive_lambda_model_fits[[counter]] = local_fit  
        
                
                if(print_info) {
                    print('...Fit...')
                    print(local_fit$par)
                    print(nhp$get_fit_standard_errors(local_fit))
                    print(local_fit$convergence)
                    print(local_fit$value)
                }
            }
        }
    }
}
```

```
## [1] ""
## [1] "constant"
## [1] "constant"
## [1] "constant"
## [1] "theta[1]+0+0"
## [1] 30
## [1] 1
## [1] "...Fit..."
## [1] 30
## [1] "Invalid standard errors produced: Use a more advanced method or improve the fit"
## [1] NA
## [1] 0
## [1] 0
## [1] ""
## [1] "constant"
## [1] "constant"
## [1] "cluster"
## [1] "theta[1]+0+theta[1 + 0 + 1]*exp((tlast - t)*abs(theta[1 + 0 + 2]))"
## [1]  30   1 100
## [1]   1   1 100
## [1] "...Fit..."
## [1] 23.912387  2.613634 18.470500
## [1]  3.539711  4.002843 49.593359
## [1] 0
## [1] -1509.057
## [1] ""
## [1] "constant"
## [1] "single_freq"
## [1] "constant"
## [1] "theta[1]+theta[1 + 1]*sin(2*pi*(t - theta[1 + 2]))+0"
## [1] 30.0  5.0  0.1
## [1] 1.0 1.0 0.1
## [1] "...Fit..."
## [1] 25.620071  7.234785  1.241399
## [1] 0.98689427 1.37676321 0.03024506
## [1] 0
## [1] -1522.613
## [1] ""
## [1] "constant"
## [1] "single_freq"
## [1] "cluster"
## [1] "theta[1]+theta[1 + 1]*sin(2*pi*(t - theta[1 + 2]))+theta[1 + 2 + 1]*exp((tlast - t)*abs(theta[1 + 2 + 2]))"
## [1]  30.0   5.0   0.1   1.0 100.0
## [1]   1.0   1.0   0.1   1.0 100.0
## [1] "...Fit..."
## [1]  25.7095968   7.2481340   2.2411091  -0.4763407 107.9284131
## [1]   1.3268795   1.3861719   0.0302362   3.6903078 705.7118914
## [1] 0
## [1] -1522.622
## [1] ""
## [1] "constant"
## [1] "double_freq"
## [1] "constant"
## [1] "theta[1]+theta[1 + 1]*sin(2*pi*(t - theta[1 + 2])) + theta[1 + 3]*sin(4*pi*(t - theta[1 + 4]))+0"
## [1] 30.0  5.0  0.1  1.0  0.1
## [1] 1.0 1.0 0.1 1.0 0.1
## [1] "...Fit..."
## [1] 25.6185643  7.3302003  1.2386166 -0.7088675  1.0531344
## [1] 0.9869049 1.4053530 0.0301801 1.3636757 0.1559097
## [1] 0
## [1] -1522.75
## [1] ""
## [1] "constant"
## [1] "double_freq"
## [1] "cluster"
## [1] "theta[1]+theta[1 + 1]*sin(2*pi*(t - theta[1 + 2])) + theta[1 + 3]*sin(4*pi*(t - theta[1 + 4]))+theta[1 + 4 + 1]*exp((tlast - t)*abs(theta[1 + 4 + 2]))"
## [1]  30.0   5.0   0.1   1.0   0.1   1.0 100.0
## [1]   1.0   1.0   0.1   1.0   0.1   1.0 100.0
## [1] "...Fit..."
## [1]  25.7207472   7.3517751   2.2384567   0.7134665   2.8042988  -0.5078305
## [7] 102.1001012
## [1]   1.33883649   1.41622172   0.03013218   1.36385721   0.15535246
## [6]   3.63079381 651.07319230
## [1] 0
## [1] -1522.759
## [1] ""
## [1] "constant"
## [1] "sawtooth"
## [1] "constant"
## [1] "theta[1]+theta[1 + 1]*abs(2/pi*asin(cos(pi*(t-theta[1+2]))))+0"
## [1] 30.0  5.0  0.5
## [1] 1.0 1.0 0.1
## [1] "...Fit..."
## [1] 16.528625 18.178177  4.510649
## [1] 1.7327549 3.3706554 0.0190393
## [1] 0
## [1] -1523.203
## [1] ""
## [1] "constant"
## [1] "sawtooth"
## [1] "cluster"
## [1] "theta[1]+theta[1 + 1]*abs(2/pi*asin(cos(pi*(t-theta[1+2]))))+theta[1 + 2 + 1]*exp((tlast - t)*abs(theta[1 + 2 + 2]))"
## [1]  30.0   5.0   0.5   1.0 100.0
## [1]   1.0   1.0   0.1   1.0 100.0
## [1] "...Fit..."
## [1]  16.575011  18.218146   4.510615  -0.342315 101.724320
## [1]   1.91989921   3.39888263   0.01890871   3.81437116 953.55636316
## [1] 0
## [1] -1523.207
## [1] ""
## [1] "soi"
## [1] "constant"
## [1] "constant"
## [1] "theta[1] + theta[2]*CI_annual_fun$soi(floor((t - 1985)%%31 + 1985))+0+0"
## [1] 30.0  0.1
## [1] 1.0 0.1
## [1] "...Fit..."
## [1] 25.7151265  0.2489929
## [1] 1.0022394 0.1322907
## [1] 0
## [1] -1510.571
## [1] ""
## [1] "soi"
## [1] "constant"
## [1] "cluster"
## [1] "theta[1] + theta[2]*CI_annual_fun$soi(floor((t - 1985)%%31 + 1985))+0+theta[2 + 0 + 1]*exp((tlast - t)*abs(theta[2 + 0 + 2]))"
## [1]  30.0   0.1   1.0 100.0
## [1]   1.0   0.1   1.0 100.0
## [1] "...Fit..."
## [1] 24.4545966  0.2428883  2.0854930 16.5631593
## [1]  3.5370332  0.1329015  4.1135718 49.8887106
## [1] 0
## [1] -1510.729
## [1] ""
## [1] "soi"
## [1] "single_freq"
## [1] "constant"
## [1] "theta[1] + theta[2]*CI_annual_fun$soi(floor((t - 1985)%%31 + 1985))+theta[2 + 1]*sin(2*pi*(t - theta[2 + 2]))+0"
## [1] 30.0  0.1  5.0  0.1
## [1] 1.0 0.1 1.0 0.1
## [1] "...Fit..."
## [1] 25.9005557  0.2362148  7.2213004  3.2437517
## [1] 1.00932920 0.12976579 1.38036935 0.03009157
## [1] 0
## [1] -1524.28
## [1] ""
## [1] "soi"
## [1] "single_freq"
## [1] "cluster"
## [1] "theta[1] + theta[2]*CI_annual_fun$soi(floor((t - 1985)%%31 + 1985))+theta[2 + 1]*sin(2*pi*(t - theta[2 + 2]))+theta[2 + 2 + 1]*exp((tlast - t)*abs(theta[2 + 2 + 2]))"
## [1]  30.0   0.1   5.0   0.1   1.0 100.0
## [1]   1.0   0.1   1.0   0.1   1.0 100.0
## [1] "...Fit..."
## [1] 26.1244978  0.2394468  7.2638994  0.2433555 -0.9768999 84.6820330
## [1]   1.4020620   0.1300174   1.3885188   0.0299871   3.5792633 355.7397232
## [1] 0
## [1] -1524.318
## [1] ""
## [1] "soi"
## [1] "double_freq"
## [1] "constant"
## [1] "theta[1] + theta[2]*CI_annual_fun$soi(floor((t - 1985)%%31 + 1985))+theta[2 + 1]*sin(2*pi*(t - theta[2 + 2])) + theta[2 + 3]*sin(4*pi*(t - theta[2 + 4]))+0"
## [1] 30.0  0.1  5.0  0.1  1.0  0.1
## [1] 1.0 0.1 1.0 0.1 1.0 0.1
## [1] "...Fit..."
## [1] 25.8968822  0.2338647  7.2890980  1.2412125  0.5732028  1.2794044
## [1] 1.00924414 0.13031784 1.40273146 0.03033195 1.36923636 0.19251035
## [1] 0
## [1] -1524.367
## [1] ""
## [1] "soi"
## [1] "double_freq"
## [1] "cluster"
## [1] "theta[1] + theta[2]*CI_annual_fun$soi(floor((t - 1985)%%31 + 1985))+theta[2 + 1]*sin(2*pi*(t - theta[2 + 2])) + theta[2 + 3]*sin(4*pi*(t - theta[2 + 4]))+theta[2 + 4 + 1]*exp((tlast - t)*abs(theta[2 + 4 + 2]))"
## [1]  30.0   0.1   5.0   0.1   1.0   0.1   1.0 100.0
## [1]   1.0   0.1   1.0   0.1   1.0   0.1   1.0 100.0
## [1] "...Fit..."
## [1]  26.0761095   0.2364274   7.3164626   0.2406297  -0.5666666   3.0315765
## [7]  -0.9896394 118.7895757
## [1]   1.43054902   0.13070187   1.41385018   0.03025118   1.36935903
## [6]   0.19479888   3.77703482 561.27474764
## [1] 0
## [1] -1524.401
## [1] ""
## [1] "soi"
## [1] "sawtooth"
## [1] "constant"
## [1] "theta[1] + theta[2]*CI_annual_fun$soi(floor((t - 1985)%%31 + 1985))+theta[2 + 1]*abs(2/pi*asin(cos(pi*(t-theta[2+2]))))+0"
## [1] 30.0  0.1  5.0  0.5
## [1] 1.0 0.1 1.0 0.1
## [1] "...Fit..."
## [1] 16.8092174  0.2408028 18.1988727  2.5125716
## [1] 1.7503485 0.1294330 3.3766297 0.0253866
## [1] 0
## [1] -1524.947
## [1] ""
## [1] "soi"
## [1] "sawtooth"
## [1] "cluster"
## [1] "theta[1] + theta[2]*CI_annual_fun$soi(floor((t - 1985)%%31 + 1985))+theta[2 + 1]*abs(2/pi*asin(cos(pi*(t-theta[2+2]))))+theta[2 + 2 + 1]*exp((tlast - t)*abs(theta[2 + 2 + 2]))"
## [1]  30.0   0.1   5.0   0.5   1.0 100.0
## [1]   1.0   0.1   1.0   0.1   1.0 100.0
## [1] "...Fit..."
## [1]  16.9141790   0.2434587  18.2708426   1.5123522  -0.8406950 126.6592605
## [1] 2.181083e+00 1.305451e-01 3.442662e+00 2.524801e-02 4.600086e+00
## [6] 1.146569e+03
## [1] 0
## [1] -1524.972
```
