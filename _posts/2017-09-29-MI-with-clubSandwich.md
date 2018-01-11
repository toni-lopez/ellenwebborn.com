---
layout: post
title: Pooling clubSandwich results across multiple imputations
date: September 29, 2017
tags: [sandwiches, R]
permalink: MI-with-clubSandwich
---

A colleague recently asked me about how to apply cluster-robust hypothesis tests and confidence intervals, as calculated with the [clubSandwich package](https://CRAN.R-project.org/package=clubSandwich), when dealing with multiply imputed datasets.
Standard methods (i.e., Rubin's rules) for pooling estimates from multiple imputed datasets are developed under the assumption that the full-data estimates are approximately normally distributed. However, this might not be reasonable when working with test statistics based on cluster-robust variance estimators, which can be imprecise when the number of clusters is small or the design matrix of predictors is unbalanced in certain ways. [Barnard and Rubin (1999)](https://doi.org/10.1093/biomet/86.4.948) proposed a small-sample correction for tests and confidence intervals based on multiple imputed datasets. In this post, I'll show how to implement their technique using the output of `clubSandwich`, with multiple imputations generated using the [`mice` package](https://cran.r-project.org/package=mice). 

### Setup

To begin, let me create missingness in a dataset containing multiple clusters of observations:


{% highlight r %}
library(mlmRev)
library(mice)
library(dplyr)

data(bdf)

bdf <- bdf %>%
  select(schoolNR, IQ.verb, IQ.perf, sex, ses, langPRET, aritPRET, aritPOST) %>%
  mutate(
    schoolNR = factor(schoolNR),
    sex = as.numeric(sex)
    ) %>%
  filter(as.numeric(schoolNR) <= 30) %>%
  droplevels()

bdf_missing <- 
  bdf %>% 
  select(-schoolNR) %>%
  ampute(run = TRUE)

bdf_missing <- 
  bdf_missing$amp %>%
  mutate(schoolNR = bdf$schoolNR)

summary(bdf_missing)
{% endhighlight %}



{% highlight text %}
##     IQ.verb         IQ.perf            sex             ses       
##  Min.   : 4.00   Min.   : 5.333   Min.   :1.000   Min.   :10.00  
##  1st Qu.:10.50   1st Qu.: 9.333   1st Qu.:1.000   1st Qu.:20.00  
##  Median :11.50   Median :10.667   Median :1.000   Median :28.00  
##  Mean   :11.68   Mean   :10.741   Mean   :1.473   Mean   :28.81  
##  3rd Qu.:13.00   3rd Qu.:12.333   3rd Qu.:2.000   3rd Qu.:38.00  
##  Max.   :18.00   Max.   :16.667   Max.   :2.000   Max.   :50.00  
##  NA's   :31      NA's   :34       NA's   :36      NA's   :36     
##     langPRET        aritPRET        aritPOST        schoolNR  
##  Min.   :15.00   Min.   : 1.00   Min.   : 2.00   40     : 35  
##  1st Qu.:29.00   1st Qu.: 9.00   1st Qu.:12.00   54     : 31  
##  Median :34.00   Median :11.00   Median :18.00   55     : 30  
##  Mean   :33.72   Mean   :11.54   Mean   :17.75   38     : 28  
##  3rd Qu.:39.00   3rd Qu.:14.00   3rd Qu.:23.00   1      : 25  
##  Max.   :48.00   Max.   :20.00   Max.   :30.00   18     : 24  
##  NA's   :48      NA's   :36      NA's   :21      (Other):354
{% endhighlight %}

Now I'll use `mice` to create 10 multiply imputed datasets:


{% highlight r %}
Impute_bdf <- mice(bdf_missing, m=10, meth="norm.nob", seed=24)
{% endhighlight %}

Am I imputing while ignoring the hierarchical structure of the data? Yes, yes I am. Is this is a good way to do imputation? Probably not. But this is a quick and dirty example, so we're going to have to live with it. 

### Model

Suppose that the goal of our analysis is to estimate the coefficients of the following regression model:

$$
\text{aritPOST}_{ij} = \beta_0 + \beta_1 \text{aritPRET}_{ij} + \beta_2 \text{langPRET}_{ij} + \beta_3 \text{sex}_{ij} + \beta_4 \text{SES}_{ij} + e_{ij},
$$

where $$i$$ indexes students and $$j$$ indexes schools, and where we allow for the possibility that errors from the same cluster are correlated in an unspecified way. With complete data, we could estimate the model by ordinary least squares and then use `clubSandwich` to get standard errors that are robust to within-cluster dependence and heteroskedasticity. The code for this is as follows:


{% highlight r %}
library(clubSandwich)
lm_full <- lm(aritPOST ~ aritPRET + langPRET + sex + ses, data = bdf)
coef_test(lm_full, cluster = bdf$schoolNR, vcov = "CR2")
{% endhighlight %}



{% highlight text %}
##          Coef Estimate     SE d.f. p-val (Satt) Sig.
## 1 (Intercept)  -2.1921 1.3484 22.9       0.1177     
## 2    aritPRET   1.0053 0.0833 23.4       <0.001  ***
## 3    langPRET   0.2758 0.0294 24.1       <0.001  ***
## 4         sex  -1.2040 0.4706 23.8       0.0173    *
## 5         ses   0.0233 0.0266 20.5       0.3909
{% endhighlight %}

If cluster dependence were no concern, we could simply use the model-based standard errors and test statistics. The `mice` package provides functions that will fit the model to each imputed dataset and then combine them by Rubin's rules. The code is simply:


{% highlight r %}
with(data = Impute_bdf, 
     lm(aritPOST ~ aritPRET + langPRET + sex + ses)
     ) %>%
  pool(method = "Rubin") %>%
  summary() %>%
  round(3)
{% endhighlight %}



{% highlight text %}
##                est    se      t       df Pr(>|t|)  lo 95  hi 95 nmis   fmi
## (Intercept) -2.003 1.118 -1.791 1529.588    0.073 -4.197  0.191   NA 0.078
## aritPRET     1.003 0.067 15.045 3508.483    0.000  0.873  1.134   36 0.051
## langPRET     0.290 0.035  8.217 4142.874    0.000  0.221  0.359   48 0.047
## sex         -1.524 0.402 -3.794  984.127    0.000 -2.313 -0.736   36 0.097
## ses          0.021 0.019  1.097 2269.299    0.273 -0.016  0.058   36 0.064
##             lambda
## (Intercept)  0.077
## aritPRET     0.051
## langPRET     0.047
## sex          0.096
## ses          0.063
{% endhighlight %}

However, this approach ignores the possibility of correlation in the errors of units in the same cluster, which is clearly a concern in this dataset:

{% highlight r %}
# ratio of CRVE to conventional variance estimates
diag(vcovCR(lm_full, cluster = bdf$schoolNR, type = "CR2")) / 
  diag(vcov(lm_full))
{% endhighlight %}



{% highlight text %}
## (Intercept)    aritPRET    langPRET         sex         ses 
##   1.5296837   1.5493134   0.6938735   1.4567650   2.0053186
{% endhighlight %}

So we need a way to pool results based on the cluster-robust variance estimators, while also accounting for the relatively small number of clusters in this dataset. 

### Barnard & Rubin (1999)

[Barnard and Rubin (1999)](https://doi.org/10.1093/biomet/86.4.948) proposed a small-sample correction for tests and confidence intervals based on multiple imputed datasets that seems to work in this context. Rather than using large-sample normal approximations for inference, they derive an approximate degrees-of-freedom that combines uncertainty in the standard errors calculated from each imputed dataset with between-imputation uncertainty. The method is as follows. 

Suppose that we have $$m$$ imputed datasets. Let $$\hat\beta_{(j)}$$ be the estimated regression coefficient from imputed dataset $$j$$, with (in this case cluster-robust) sampling variance estimate $$V_{(j)}$$. Further, let $$\eta_{(j)}$$ be the degrees of freedom corresponding to $$V_{(j)}$$. To combine these estimates, calculate the averages across multiply imputed datasets:

$$
\bar\beta = \frac{1}{m}\sum_{j=1}^m \hat\beta_{(j)}, \qquad \bar{V} = \frac{1}{m}\sum_{j=1}^m V_{(j)}, \qquad \bar\eta = \frac{1}{m}\sum_{j=1}^m \eta_{(j)}.
$$

Also calculate the between-imputation variance 

$$
B = \frac{1}{m - 1} \sum_{j=1}^m \left(\hat\beta_{(j)} - \bar\beta\right)^2
$$

And then combine the between- and within- variance estimates using Rubin's rules:

$$
V_{total} = \bar{V} + \frac{m + 1}{m} B.
$$

The degrees of freedom associated with $$V_{total}$$ modify the estimated complete-data degrees of freedom $$\bar\eta$$ using quantities that depend on the fraction of missing information in a coefficient. The fraction of missing information is given by

$$
\hat\gamma_m = \frac{(m+1)B}{m V_{total}}
$$

The degrees of freedom are then given by

$$
\nu_{total} = \left(\frac{1}{\nu_m} + \frac{1}{\nu_{obs}}\right)^{-1},
$$

where

$$
\nu_m = \frac{(m - 1)}{\hat\gamma_m^2}, \quad \text{and} \quad \nu_{obs} = \frac{\bar\eta (\bar\eta + 1) (1 - \hat\gamma)}{\bar\eta + 3}.
$$

Hypothesis tests and confidence intervals are based on the approximation

$$
\frac{\bar\beta - \beta_0}{\sqrt{V_{total}}} \ \stackrel{\cdot}{\sim} \ t(\nu_{total})
$$

### Implementation 

Here is how to carry out these calculations using the results of `clubSandwich::coef_test` and a bit of `dplyr`:


{% highlight r %}
# fit results with clubSandwich standard errors

models_robust <- with(data = Impute_bdf, 
                      lm(aritPOST ~ aritPRET + langPRET + sex + ses) %>% 
                         coef_test(cluster=bdf$schoolNR, vcov="CR2")
                      ) 


# pool results with clubSandwich standard errors

robust_pooled <- 
  models_robust$analyses %>%
  
  # add coefficient names as a column
  lapply(function(x) {
    x$coef <- row.names(x)
    x
  }) %>%
  bind_rows() %>%
  as.data.frame() %>%
  
  # summarize by coefficient
  group_by(coef) %>%
  summarise(
    m = n(),
    B = var(beta),
    beta_bar = mean(beta),
    V_bar = mean(SE^2),
    eta_bar = mean(df)
  ) %>%
  
  mutate(
    
    # calculate intermediate quantities to get df
    V_total = V_bar + B * (m + 1) / m,
    gamma = ((m + 1) / m) * B / V_total,
    df_m = (m - 1) / gamma^2,
    df_obs = eta_bar * (eta_bar + 1) * (1 - gamma) / (eta_bar + 3),
    df = 1 / (1 / df_m + 1 / df_obs),
    
    # calculate summary quantities for output
    se = sqrt(V_total),
    t = beta_bar / se,
    p_val = 2 * pt(abs(t), df = df, lower.tail = FALSE),
    crit = qt(0.975, df = df),
    lo95 = beta_bar - se * crit,
    hi95 = beta_bar + se * crit
  )

robust_pooled %>%
  select(coef, est = beta_bar, se, t, df, p_val, lo95, hi95, gamma) %>%
  mutate_at(vars(est:gamma), round, 3)
{% endhighlight %}



{% highlight text %}
## # A tibble: 5 x 9
##          coef    est    se      t     df p_val   lo95   hi95 gamma
##         <chr>  <dbl> <dbl>  <dbl>  <dbl> <dbl>  <dbl>  <dbl> <dbl>
## 1 (Intercept) -2.003 1.317 -1.521 19.946 0.144 -4.752  0.745 0.055
## 2    aritPRET  1.003 0.085 11.808 20.772 0.000  0.827  1.180 0.031
## 3    langPRET  0.290 0.031  9.316 20.764 0.000  0.225  0.354 0.060
## 4         ses  0.021 0.027  0.785 18.202 0.443 -0.035  0.077 0.032
## 5         sex -1.524 0.430 -3.546 19.554 0.002 -2.423 -0.626 0.084
{% endhighlight %}



It is instructive to compare the calculated `df` to `eta_bar` and `df_m`: 

{% highlight r %}
robust_pooled %>%
  select(coef, df, df_m, eta_bar) %>%
  mutate_at(vars(df, df_m, eta_bar), round, 1)
{% endhighlight %}



{% highlight text %}
## # A tibble: 5 x 4
##          coef    df   df_m eta_bar
##         <chr> <dbl>  <dbl>   <dbl>
## 1 (Intercept)  19.9 2943.5    23.0
## 2    aritPRET  20.8 9247.5    23.3
## 3    langPRET  20.8 2506.7    24.1
## 4         ses  18.2 8646.7    20.6
## 5         sex  19.6 1289.8    23.4
{% endhighlight %}

Here, `eta_bar` is the average of the complete data degrees of freedom, and it can be seen that the total degrees of freedom are somewhat less than the average complete-data degrees of freedom. This is by construction. Further `df_m` is the conventional degrees of freedom used in multiple-imputation, which assume that the complete-data estimates are normally distributed, and in this example they are way far off. 

### Further thoughts

How well does this method perform in practice? I'm not entirely sure---I'm just trusting that Barnard and Rubin's approximation is sound and would work in this setting (I mean, they're smart people!). Are there other, better approaches? Totally possible. I have done zero literature review beyond the Barnard and Rubin paper. In any case, exploring the performance of this method (and any other alternatives) seems like it would make for a very nice student project. 

There's also the issue of how to do tests of multi-dimensional constraints (i.e., F-tests). The `clubSandwich` package implements Wald-type tests for multi-dimensional constraints, using a small-sample correction that we developed ([Tipton & Pustejovsky, 2015](http://journals.sagepub.com/doi/abs/10.3102/1076998615606099); [Pustejovsky & Tipton, 2016](http://www.tandfonline.com/doi/full/10.1080/07350015.2016.1247004)). But it would take some further thought to figure out how to handle multiply imputed data with this type of test... 