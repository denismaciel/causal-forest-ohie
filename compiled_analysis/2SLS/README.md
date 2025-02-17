2SLS
================
Denis Maciel
7/13/2019

``` r
suppressPackageStartupMessages(library(tidyverse))

source(here::here("code", "load_datasets.R"))

load_datasets()
```

Prepare the data:

  - Make treatment a numeric variable (0 for control, 1 for treatment)
  - Extract the number of household members present in the lottery.
  - Make any visit to emergency department binary and numeric

`ohp_all_ever_firstn_30sep2009`: This variable takes a value of 1 if an
individual was enrolled in any Medicaid program (including the lotteried
program, OHP Standard) between the earliest notification date in the
sample (10 March 2008) and 30 September 2009. In the analysis of the
12-month mail survey data in Finkelstein et al (2012), this variable was
used as the definition of insurance coverage in estimating the effect of
Medicaid. In the analysis in Taubman et al (201XX), this variable was
used as the definition of insurance coverage in estimating the effect of
Medicaid.

``` r
# Number of household in the list
eme_visits <- emergency %>% 
  left_join(descriptive %>% select(person_id, treatment, numhh_list)) %>% 
  left_join(state_programs %>% select(person_id, medicaid = ohp_all_ever_firstn_30sep2009)) %>% 
  mutate(n_household = dplyr::case_when(
    numhh_list == "signed self up" ~ 0L,
    numhh_list == "signed self up + 1 additional person" ~ 1L,
    numhh_list == "signed self up + 2 additional people" ~ 2L
  ))
```

    ## Joining, by = "person_id"
    ## Joining, by = "person_id"

``` r
# Treatment binary and numeric
eme_visits <- eme_visits %>% 
  mutate(treat_numeric = case_when(
    treatment == "Selected" ~ 1L,
    treatment == "Not selected" ~ 0L
  ))

eme_visits <- eme_visits %>% 
  mutate(any_visit_ed = case_when(
    any_visit_ed == "Yes" ~ 1L,
    any_visit_ed == "No" ~ 0L
  ))

eme_visits <- eme_visits %>% 
  mutate(medicaid = case_when(
    medicaid == "Enrolled" ~ 1L,
    medicaid == "NOT enrolled" ~ 0L
  ))

stopifnot(
  all(!is.na(eme_visits$n_household))
)

stopifnot(
  all(!is.na(eme_visits$treat_numeric))
)

stopifnot(
  all(!is.na(eme_visits$medicaid))
)
```

``` r
eme_visits %>% 
  lm(data = ., formula = num_visit_cens_ed ~ treat_numeric + n_household)
```

    ## 
    ## Call:
    ## lm(formula = num_visit_cens_ed ~ treat_numeric + n_household, 
    ##     data = .)
    ## 
    ## Coefficients:
    ##   (Intercept)  treat_numeric    n_household  
    ##       1.08719        0.08502       -0.59975

``` r
eme_visits %>% 
  group_by(treatment) %>% 
  summarise(mean(num_visit_cens_ed, na.rm = TRUE))
```

    ## # A tibble: 2 x 2
    ##   treatment    `mean(num_visit_cens_ed, na.rm = TRUE)`
    ##   <fct>                                          <dbl>
    ## 1 Not selected                                   1.00 
    ## 2 Selected                                       0.990

## 2SLS

First Stage Regression: Regress Medicaid enrollment on the lottery

``` r
first_stage <- lm(data = eme_visits, formula = medicaid ~ treat_numeric)

# add fitted values to dataframe
eme_visits$medicaid_hat <- first_stage$fitted.values
```

Second Stage Regression

``` r
eme_visits %>% 
  lm(data = ., formula = num_visit_cens_ed ~ medicaid_hat + n_household)
```

    ## 
    ## Call:
    ## lm(formula = num_visit_cens_ed ~ medicaid_hat + n_household, 
    ##     data = .)
    ## 
    ## Coefficients:
    ##  (Intercept)  medicaid_hat   n_household  
    ##       1.0336        0.3539       -0.5997

``` r
eme_visits %>% 
  lm(data = ., formula = any_visit_ed ~ medicaid_hat + n_household)
```

    ## 
    ## Call:
    ## lm(formula = any_visit_ed ~ medicaid_hat + n_household, data = .)
    ## 
    ## Coefficients:
    ##  (Intercept)  medicaid_hat   n_household  
    ##      0.35281       0.07859      -0.13930
