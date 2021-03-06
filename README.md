Mental_Illness.Mar20_Feb21.Rmd
================

## Project Goals

Replicate Mental Health By Political Affiliation Study by Zach Goldberg
(<https://threadreaderapp.com/thread/1248823584111439872.html>)

Zach Goldberg demonstrated that liberals are much more likely to report
having been diagnosed with a mental illness in a pew research survey
from March 2020
(<https://www.pewresearch.org/social-trends/dataset/covid-19-late-march-2020/>)

A stated limitation of Goldberg’s analysis was: ‘It’s possible that the
disparities in self-reported diagnosis are simply or partly a function
of white liberals being more likely to seek mental health evaluations. I
don’t have the data to answer this question.’

However, the data to answer that question was available in the Pew
questionnaire. In addition to asking, ‘Has a doctor or other healthcare
provider EVER told you that you have a mental health condition ?’ the
survey also asked:

1.  In the past 7 days, how often have you… Felt nervous, anxious, or on
    edge?
2.  In the past 7 days, how often have you… Felt depressed?
3.  In the past 7 days, how often have you… Felt lonely?
4.  In the past 7 days, how often have you… Felt hopeful about the
    future?
5.  In the past 7 days, how often have you… Had trouble sleeping?

Each of these was scored 1-5 with higher scores indicating more frequent
distress (D. was reversed) and aggregated in a variable titled
‘MHEALTH’.

These questions were asked again one year later in the W83 survey
(<https://www.pewresearch.org/science/dataset/american-trends-panel-wave-83/>),
providing opportunity the replicate the findings

### Specific Goals:

1.  Independently replicate Goldberg’s findings.
2.  Repeat analysis using self-reported mental health variables
    (MHEALTH).
3.  Replicate findings in another data set (W83).

## Step 1: Load the original W64 data set

``` r
mar20Dat <- haven::read_sav(file=file.path('W64_Mar20/', 'ATP W64.sav'))

#Make a working copy
mar20DF <- as.data.frame(mar20Dat)
```

## Step 2: Re-encode variables as necessary

Main Variables “F_IDEO”: Ideology Labels: value label 1 Very
conservative 2 Conservative 3 Moderate 4 Liberal 5 Very liberal

“COVID_MENTAL_W64”: “Has a doctor or other healthcare provider EVER told
you that you have a mental health condition ?”  
“MHEALTH”: “Simple additive index of 5 mental health measures”

Relevant covariates “F_ACSWEB”: “ACS Internet Usage” “F_AGECAT”: “Age
category” “F_MARITAL”: “Marital status” “F_EDUCCAT2”: “Education level
category 2” “F_INCOME”: “Family income” “F_ATTEND”: “Religious service
attendance” “F_RACETHN”: “Race-Ethnicity” “F_SEX”: “Sex”

Re-encoded so that a higher score means more of the thing measured (for
example, a higher MARITAL score means married) and variables such as
RACE become encoded as YES/NO answers to individual races (For example:
Hispanic).

``` r
#Note: Race will need to be one-hot
relCov <- c("F_ACSWEB", "F_AGECAT", "F_MARITAL", "F_EDUCCAT2", "F_INCOME",  "F_IDEO", "F_ATTEND")  #"MHEALTH", "F_RACETHN" , "F_SEX", "COVID_MENTAL_W64"

mar20Dat.curDF <- as.data.frame(mar20Dat[, c(relCov, "MHEALTH", "F_RACETHN", "F_SEX", "COVID_MENTAL_W64")])
mar20Dat.curDF[mar20Dat.curDF==99] <- as.numeric(NA)
mar20Dat.curDF <- mar20Dat.curDF[!apply(mar20Dat.curDF, 1, function(x){any(is.na(x))}),]

#One-hot race
F_RACETHN_Name <- names(attr(mar20Dat$F_RACETHN, "labels"))[match(mar20Dat.curDF$F_RACETHN, attr(mar20Dat$F_RACETHN, "labels"))]
table(F_RACETHN_Name)
```

    ## F_RACETHN_Name
    ## Black non-Hispanic           Hispanic              Other White non-Hispanic 
    ##                814               2167                590               7066

``` r
oHot_Race <- fastDummies::dummy_cols(F_RACETHN_Name, remove_selected_columns=TRUE)
colnames(oHot_Race) <- make.names(sub('\\.data_', '', colnames(oHot_Race)))

#One-hot sex
F_SEX_Name <- names(attr(mar20Dat$F_SEX, "labels"))[match(mar20Dat.curDF$F_SEX, attr(mar20Dat$F_SEX, "labels"))]
table(F_SEX_Name)
```

    ## F_SEX_Name
    ## Female   Male 
    ##   5792   4845

``` r
oHot_Sex <- fastDummies::dummy_cols(F_SEX_Name, remove_selected_columns=TRUE)
colnames(oHot_Sex) <- make.names(sub('\\.data_', '', colnames(oHot_Sex)))

oHots <- cbind(oHot_Race, oHot_Sex)

#Fix religion and marriage so higher is better
mar20Dat.curDF <- cbind(mar20Dat.curDF, oHots)
mar20Dat.curDF$F_HOW_MUCH_RELIGION <- 7-mar20Dat.curDF$F_ATTEND

mar20Dat.curDF$F_HOW_MUCH_MARRIED <- 7-mar20Dat.curDF$F_MARITAL

#Fix mental illness diagnosis so 1 is TRUE and 0 is FALSE
mar20Dat.curDF$DIAG_MENTAL_ILLNESS <- 2-mar20Dat.curDF$COVID_MENTAL_W64

#Fix MHEALTH so it varies by percentage of mental distress
mar20Dat.curDF$MENTAL_DISTRESS_PERCENT <- round(100*(mar20Dat.curDF$MHEALTH - min(mar20Dat.curDF$MHEALTH))/(max(mar20Dat.curDF$MHEALTH)- min(mar20Dat.curDF$MHEALTH)))
```

## Step 3: Replicate Diagnosed Mental Illness Results

If Zach’s observations hold up, the more liberal a person is, the more
likely they are to have been diagnosed with a mental illness.

``` r
diagMentalTable <- table(mar20Dat.curDF$DIAG_MENTAL_ILLNESS, mar20Dat.curDF$F_IDEO)
rownames(diagMentalTable) <- c('NOT.DIAGNOSED', 'DIAG.MENTAL.ILL')
colnames(diagMentalTable) <- names(attr(mar20Dat$F_IDEO, "labels"))[match(colnames(diagMentalTable), attr(mar20Dat$F_IDEO, "labels"))]
#knitr::kable(diagMentalTable)
diagMentalTable
```

    ##                  
    ##                   Very conservative Conservative Moderate Liberal Very liberal
    ##   NOT.DIAGNOSED                 713         2144     3512    1827          753
    ##   DIAG.MENTAL.ILL                83          243      606     464          292

``` r
diagMentalTable.per <- apply(diagMentalTable, 2, function(x){round(100*x/sum(x),1)})
#knitr::kable(diagMentalTable.per)
diagMentalTable.per
```

    ##                  
    ##                   Very conservative Conservative Moderate Liberal Very liberal
    ##   NOT.DIAGNOSED                89.6         89.8     85.3    79.7         72.1
    ##   DIAG.MENTAL.ILL              10.4         10.2     14.7    20.3         27.9

``` r
diagTable.per <- diagTable <- list()
for(curAge in unique(mar20Dat.curDF$F_AGECAT)) {
  ageName <- names(attr(mar20Dat.curDF$F_AGECAT, 'labels'))[attr(mar20Dat.curDF$F_AGECAT, 'labels') == curAge]
  ageName <- paste(curAge, ageName, sep='_')
  curIdx <- mar20Dat.curDF$F_AGECAT == curAge

  dMT <- table(mar20Dat.curDF[curIdx,'DIAG_MENTAL_ILLNESS'], mar20Dat.curDF[curIdx,'F_IDEO'])
  rownames(dMT) <- c('NOT.DIAGNOSED', 'DIAG.MENTAL.ILL')
  colnames(dMT) <- names(attr(mar20Dat$F_IDEO, "labels"))[match(colnames(dMT), attr(mar20Dat$F_IDEO, "labels"))]

  dMT.per <- apply(dMT, 2, function(x){round(100*x/sum(x),1)})
  
  diagTable[[ageName]] <- dMT
  diagTable.per[[ageName]] <- dMT.per
}
diagTable.per <- diagTable.per[order(names(diagTable.per))]
message('Percentage with diagnosed mentall illness by Age catagory')
```

    ## Percentage with diagnosed mentall illness by Age catagory

``` r
diagTable.per
```

    ## $`1_18-29`
    ##                  
    ##                   Very conservative Conservative Moderate Liberal Very liberal
    ##   NOT.DIAGNOSED                  84         83.5     81.2    72.4           65
    ##   DIAG.MENTAL.ILL                16         16.5     18.8    27.6           35
    ## 
    ## $`2_30-49`
    ##                  
    ##                   Very conservative Conservative Moderate Liberal Very liberal
    ##   NOT.DIAGNOSED                85.2         87.4       81    76.2           67
    ##   DIAG.MENTAL.ILL              14.8         12.6       19    23.8           33
    ## 
    ## $`3_50-64`
    ##                  
    ##                   Very conservative Conservative Moderate Liberal Very liberal
    ##   NOT.DIAGNOSED                90.1         88.7     86.6    82.8         72.2
    ##   DIAG.MENTAL.ILL               9.9         11.3     13.4    17.2         27.8
    ## 
    ## $`4_65+`
    ##                  
    ##                   Very conservative Conservative Moderate Liberal Very liberal
    ##   NOT.DIAGNOSED                93.4         94.8     92.3    86.2         88.7
    ##   DIAG.MENTAL.ILL               6.6          5.2      7.7    13.8         11.3

``` r
mar20Dat.curDF$F_IDEO_Name <- names(attr(mar20Dat$F_IDEO, "labels"))[match(mar20Dat.curDF$F_IDEO, attr(mar20Dat$F_IDEO, "labels"))]
means <- tapply(mar20Dat.curDF$DIAG_MENTAL_ILLNESS, mar20Dat.curDF$F_IDEO_Name, mean, na.rm=TRUE)
means <- sort(means)

plot(means*100, main='Mental Illness by Ideology', xaxt = "n", xlab = 'Ideology', ylab = "% Diagnosed Mentally Ill", type = 'o')
axis(1, at = 1:length(means), labels = names(means))
```

![](Mental_Illness.Mar20_Feb21_files/figure-gfm/DiagMentalIll.plot-1.png)<!-- -->

## Step 4: Repeat analysis using mental health score.

MHEALTH is an measure of mental distress over the last 7 days. It varies
between 5 (least distressed) and 20 (most distressed). This is converted
to a percentage of the maximum amount of mental distress

``` r
message('Distribution of mental distress scores')
```

    ## Distribution of mental distress scores

``` r
summary(mar20Dat.curDF$MHEALTH)
```

    ##    Min. 1st Qu.  Median    Mean 3rd Qu.    Max. 
    ##   5.000   6.000   8.000   8.991  11.000  20.000

``` r
message('Distribution of mental distress percentages of max')
```

    ## Distribution of mental distress percentages of max

``` r
summary(mar20Dat.curDF$MENTAL_DISTRESS_PERCENT)
```

    ##    Min. 1st Qu.  Median    Mean 3rd Qu.    Max. 
    ##    0.00    7.00   20.00   26.62   40.00  100.00

``` r
mar20Dat.curDF$F_IDEO_Name <- names(attr(mar20Dat$F_IDEO, "labels"))[match(mar20Dat.curDF$F_IDEO, attr(mar20Dat$F_IDEO, "labels"))]
means <- tapply(mar20Dat.curDF$MENTAL_DISTRESS_PERCENT, mar20Dat.curDF$F_IDEO_Name, mean, na.rm=TRUE)
mar20Dat.curDF$F_IDEO_Name <- factor(mar20Dat.curDF$F_IDEO_Name,levels=names(means)[order(means)])

p <- ggplot(mar20Dat.curDF, aes(x = F_IDEO_Name, y = MENTAL_DISTRESS_PERCENT))
p <- p + geom_boxplot()
p <- p + stat_summary(fun=mean, shape=1,col='red', geom='point')
print(p)
```

![](Mental_Illness.Mar20_Feb21_files/figure-gfm/PlotMHscoress-1.png)<!-- -->
Unlike the diagnosed mental illness (which seems to be following a
hockey stick shape), the mental distress score seems to follow a linear
relationship. However, the same pattern holds: the more liberal you are,
the more symptoms of mental illness you report.

## Step 5: Control for covariates

One might expect that someone who has a very low income might naturally
suffer from greater mental distress due to poverty as well as identify
with a more liberal ideology since liberals tend to want to provide more
services to low income people. Therefore, it will be necessary to
control for different covariates such as income.

``` r
eq1 <- sub('F_MARITAL', 'F_HOW_MUCH_MARRIED', sub('F_ATTEND', 'F_HOW_MUCH_RELIGION', paste(relCov, collapse=' + ')))
eq2 <- paste(colnames(oHots)[!colnames(oHots) %in% c('Other', 'Male')], collapse=' + ')
fullEq.str <- paste0('MENTAL_DISTRESS_PERCENT ~ ', eq1, " + ", eq2, " + DIAG_MENTAL_ILLNESS")

message('Prediction of MENTAL_DISTRESS_PERCENT aggregate variable')
```

    ## Prediction of MENTAL_DISTRESS_PERCENT aggregate variable

``` r
message(fullEq.str)
```

    ## MENTAL_DISTRESS_PERCENT ~ F_ACSWEB + F_AGECAT + F_HOW_MUCH_MARRIED + F_EDUCCAT2 + F_INCOME + F_IDEO + F_HOW_MUCH_RELIGION + Black.non.Hispanic + Hispanic + White.non.Hispanic + Female + DIAG_MENTAL_ILLNESS

``` r
fullEq <- as.formula(fullEq.str)

fullReg <- glm(formula = fullEq, data = mar20Dat.curDF)
summary(fullReg)
```

    ## 
    ## Call:
    ## glm(formula = fullEq, data = mar20Dat.curDF)
    ## 
    ## Deviance Residuals: 
    ##     Min       1Q   Median       3Q      Max  
    ## -49.674  -14.566   -3.446   11.312   87.923  
    ## 
    ## Coefficients:
    ##                     Estimate Std. Error t value Pr(>|t|)    
    ## (Intercept)         22.93497    1.87589  12.226  < 2e-16 ***
    ## F_ACSWEB             2.93921    1.04459   2.814 0.004906 ** 
    ## F_AGECAT            -2.44557    0.21189 -11.542  < 2e-16 ***
    ## F_HOW_MUCH_MARRIED  -0.50492    0.10981  -4.598 4.31e-06 ***
    ## F_EDUCCAT2          -0.06462    0.15024  -0.430 0.667103    
    ## F_INCOME            -0.38749    0.10200  -3.799 0.000146 ***
    ## F_IDEO               2.91984    0.20409  14.307  < 2e-16 ***
    ## F_HOW_MUCH_RELIGION -0.89134    0.12731  -7.001 2.69e-12 ***
    ## Black.non.Hispanic  -1.91999    1.08973  -1.762 0.078117 .  
    ## Hispanic             0.84935    0.93098   0.912 0.361621    
    ## White.non.Hispanic   0.97761    0.85978   1.137 0.255543    
    ## Female               5.49304    0.39586  13.876  < 2e-16 ***
    ## DIAG_MENTAL_ILLNESS 13.73272    0.54652  25.128  < 2e-16 ***
    ## ---
    ## Signif. codes:  0 '***' 0.001 '**' 0.01 '*' 0.05 '.' 0.1 ' ' 1
    ## 
    ## (Dispersion parameter for gaussian family taken to be 394.6355)
    ## 
    ##     Null deviance: 5028545  on 10636  degrees of freedom
    ## Residual deviance: 4192607  on 10624  degrees of freedom
    ## AIC: 93789
    ## 
    ## Number of Fisher Scoring iterations: 2

``` r
regPred <- predict(fullReg)
r2 <- cor(regPred, mar20Dat.curDF$MHEALTH)^2
message(paste('R2 =', round(r2,3)))
```

    ## R2 = 0.166

If you’re not used to looking at results like these they can be
intimidating. Here’s the important parts:

Coefficients: Estimate Std. Error t value Pr(>\|t\|)  
F_AGECAT -2.44557 0.21189 -11.542 \< 2e-16 \*\*\*

The first column is the variable name. The column labeled ‘Pr(>\|t\|)’
is a p-Value indicating how significant that variable is for predicting
MENTAL_DISTRESS_PERCENT; In general you can ignore anything with a
p-Value \> 0.05 and the things with very low p-Values (\< 2e-16) have
the most certain effects. ‘Estimate’ is the effect size; -2.4 means that
for each step from the age group 18-29 to the age group 65+ reduces the
percentage of mental distress by 2.4% after accounting for all of the
other variables included in the model.

The R2 value is a measure of goodness of fit. 0.166 means that about
16.6% of the variance in MENTAL_DISTRESS_PERCENT is explained by this
model. In other words, about 83.4% of a person’s mental distress is
caused by other things.

A powerful tool for removing variables that are caused by other
variables is LASSO regression. For example if the effect of age was due
to being married or not, LASSO would remove the age variable.

``` r
x <- mar20Dat.curDF[ , strsplit(as.character(fullEq)[3], ' \\+ ')[[1]]]
y <- mar20Dat.curDF$MENTAL_DISTRESS_PERCENT 
lasso.cv <- cv.glmnet(x=as.matrix(x), y=y)

message('These parameters survive LASSO regression to predict MHEALTH')
```

    ## These parameters survive LASSO regression to predict MHEALTH

``` r
extract.coef(lasso.cv)
```

    ##                            Value SE         Coefficient
    ## (Intercept)         23.852363728 NA         (Intercept)
    ## F_ACSWEB             2.451726484 NA            F_ACSWEB
    ## F_AGECAT            -2.358040667 NA            F_AGECAT
    ## F_HOW_MUCH_MARRIED  -0.473952029 NA  F_HOW_MUCH_MARRIED
    ## F_EDUCCAT2          -0.009555756 NA          F_EDUCCAT2
    ## F_INCOME            -0.376395660 NA            F_INCOME
    ## F_IDEO               2.841406376 NA              F_IDEO
    ## F_HOW_MUCH_RELIGION -0.870450762 NA F_HOW_MUCH_RELIGION
    ## Black.non.Hispanic  -2.275518211 NA  Black.non.Hispanic
    ## White.non.Hispanic   0.055247812 NA  White.non.Hispanic
    ## Female               5.340753075 NA              Female
    ## DIAG_MENTAL_ILLNESS 13.608577998 NA DIAG_MENTAL_ILLNESS

``` r
predVal <- predict(lasso.cv, newx=as.matrix(x))
r2 <- cor(predVal, y)^2
message(paste('R2 =', round(r2,3)))
```

    ## R2 = 0.162

In these LASSO results you can see

-2.358040667 NA F_AGECAT which essentially repeate the previous results,
where each step from the age group 18-29 to the age group 65+ reduces
the percentage of mental distress by \~2.4% after accounting for all of
the other variables included in the model.

2.841406376 NA F_IDEO Shows that after controlling for the other
variables, each step from “1: Very conservative” to “5: Very liberal”
increases mental distress by about 2.8%

Note that this analysis is limited in that it’s using linear models,
which might not be appropriate. For example F_HOW_MUCH_MARRIED considers
‘Widowed’ as one point less married than ‘Separated’ which is 3 points
less married than ‘Married’. In other words: there is plenty of room for
improvement.

## Step 6: Replicate on another data set

For follow-up, use W83 from Feb 21. That also asks for political party
and signs of mental health stress.

``` r
feb21Dat <- haven::read_sav(file=file.path('W83_Feb21/', 'ATP W83.sav'))
```

``` r
#Note: Race will need to be one-hot
relCov <- c("F_AGECAT", "F_MARITAL", "F_EDUCCAT2", "F_INC_SDT1",  "F_IDEO", "F_ATTEND")  #"MHEALTH", "F_RACETHN" , "F_SEX", "COVID_MENTAL_W64"

feb21DF <- as.data.frame(feb21Dat[, c(relCov, "MHEALTH_W83", "F_RACECMB", "F_GENDER")])
feb21DF[feb21DF==99] <- as.numeric(NA)
feb21DF <- feb21DF[!apply(feb21DF, 1, function(x){any(is.na(x))}),]

#One-hot race
F_RACETHN_Name <- names(attr(feb21Dat$F_RACECMB, "labels"))[match(feb21DF$F_RACECMB, attr(feb21Dat$F_RACECMB, "labels"))]
table(F_RACETHN_Name)
```

    ## F_RACETHN_Name
    ##   Asian or Asian-American Black or African-American                Mixed Race 
    ##                       322                       886                       421 
    ##        Or some other race                     White 
    ##                       361                      7280

``` r
oHot_Race <- fastDummies::dummy_cols(F_RACETHN_Name, remove_selected_columns=TRUE)
colnames(oHot_Race) <- make.names(sub('\\.data_', '', colnames(oHot_Race)))

#One-hot sex
F_SEX_Name <- names(attr(feb21Dat$F_GENDER, "labels"))[match(feb21DF$F_GENDER, attr(feb21Dat$F_GENDER, "labels"))]
table(F_SEX_Name)
```

    ## F_SEX_Name
    ##             A man           A woman In some other way 
    ##              4223              4997                50

``` r
oHot_Sex <- fastDummies::dummy_cols(F_SEX_Name, remove_selected_columns=TRUE)
colnames(oHot_Sex) <- make.names(sub('\\.data_', '', colnames(oHot_Sex)))

oHots <- cbind(oHot_Race, oHot_Sex)

#Fix religion and marriage so higher is better
feb21DF <- cbind(feb21DF, oHots)
feb21DF$F_HOW_MUCH_RELIGION <- 7-feb21DF$F_ATTEND

feb21DF$F_HOW_MUCH_MARRIED <- 7-feb21DF$F_MARITAL

#Fix MHEALTH so it varies by percentage of mental distress
feb21DF$MENTAL_DISTRESS_PERCENT <- round(100*(feb21DF$MHEALTH_W83 - min(feb21DF$MHEALTH_W83))/(max(feb21DF$MHEALTH_W83)- min(feb21DF$MHEALTH_W83)))
```

``` r
eq1 <- sub('F_MARITAL', 'F_HOW_MUCH_MARRIED', sub('F_ATTEND', 'F_HOW_MUCH_RELIGION', paste(relCov, collapse=' + ')))
eq2 <- paste(colnames(oHots)[!colnames(oHots) %in% c('Other', 'Male')], collapse=' + ')
fullEq.str <- paste0('MENTAL_DISTRESS_PERCENT ~ ', eq1, " + ", eq2)

message('Prediction of MENTAL_DISTRESS_PERCENT aggregate variable')
```

    ## Prediction of MENTAL_DISTRESS_PERCENT aggregate variable

``` r
message(fullEq.str)
```

    ## MENTAL_DISTRESS_PERCENT ~ F_AGECAT + F_HOW_MUCH_MARRIED + F_EDUCCAT2 + F_INC_SDT1 + F_IDEO + F_HOW_MUCH_RELIGION + Asian.or.Asian.American + Black.or.African.American + Mixed.Race + Or.some.other.race + White + A.man + A.woman + In.some.other.way

``` r
fullEq <- as.formula(fullEq.str)

fullReg <- glm(formula = fullEq, data = feb21DF)
summary(fullReg)
```

    ## 
    ## Call:
    ## glm(formula = fullEq, data = feb21DF)
    ## 
    ## Deviance Residuals: 
    ##     Min       1Q   Median       3Q      Max  
    ## -44.278  -14.921   -4.285   11.291   82.252  
    ## 
    ## Coefficients: (2 not defined because of singularities)
    ##                            Estimate Std. Error t value Pr(>|t|)    
    ## (Intercept)                50.79058    3.13100  16.222  < 2e-16 ***
    ## F_AGECAT                   -3.16524    0.22921 -13.809  < 2e-16 ***
    ## F_HOW_MUCH_MARRIED         -1.00491    0.12042  -8.345  < 2e-16 ***
    ## F_EDUCCAT2                 -0.65232    0.16230  -4.019 5.88e-05 ***
    ## F_INC_SDT1                 -0.83212    0.08262 -10.071  < 2e-16 ***
    ## F_IDEO                      1.65924    0.22099   7.508 6.55e-14 ***
    ## F_HOW_MUCH_RELIGION        -0.89070    0.13853  -6.430 1.34e-10 ***
    ## Asian.or.Asian.American    -3.59257    1.17439  -3.059  0.00223 ** 
    ## Black.or.African.American  -3.19797    0.75345  -4.244 2.21e-05 ***
    ## Mixed.Race                  0.73690    1.02749   0.717  0.47328    
    ## Or.some.other.race         -3.38646    1.11432  -3.039  0.00238 ** 
    ## White                            NA         NA      NA       NA    
    ## A.man                     -11.48650    2.91435  -3.941 8.16e-05 ***
    ## A.woman                    -5.82944    2.90478  -2.007  0.04480 *  
    ## In.some.other.way                NA         NA      NA       NA    
    ## ---
    ## Signif. codes:  0 '***' 0.001 '**' 0.01 '*' 0.05 '.' 0.1 ' ' 1
    ## 
    ## (Dispersion parameter for gaussian family taken to be 414.893)
    ## 
    ##     Null deviance: 4334065  on 9269  degrees of freedom
    ## Residual deviance: 3840665  on 9257  degrees of freedom
    ## AIC: 82202
    ## 
    ## Number of Fisher Scoring iterations: 2

``` r
regPred <- predict(fullReg)
r2 <- cor(regPred, feb21DF$MENTAL_DISTRESS_PERCENT)^2
message(paste('R2 =', round(r2,3)))
```

    ## R2 = 0.114

Note that this analysis has an R2 value \~5% lower than the previous
model. This is probably due to the missing question about diagnosed
mental illness. If you remove that variable from the previous model,
that R2 value drops to around 0.117.

``` r
x <- feb21DF[ , strsplit(as.character(fullEq)[3], ' \\+ ')[[1]]]
y <- feb21DF$MENTAL_DISTRESS_PERCENT 
lasso.cv <- cv.glmnet(x=as.matrix(x), y=y)

message('These parameters survive LASSO regression to predict MHEALTH')
```

    ## These parameters survive LASSO regression to predict MHEALTH

``` r
extract.coef(lasso.cv)
```

    ##                              Value SE             Coefficient
    ## (Intercept)             41.7234987 NA             (Intercept)
    ## F_AGECAT                -3.1520516 NA                F_AGECAT
    ## F_HOW_MUCH_MARRIED      -1.0000203 NA      F_HOW_MUCH_MARRIED
    ## F_EDUCCAT2              -0.6418463 NA              F_EDUCCAT2
    ## F_INC_SDT1              -0.8301712 NA              F_INC_SDT1
    ## F_IDEO                   1.6470000 NA                  F_IDEO
    ## F_HOW_MUCH_RELIGION     -0.8873944 NA     F_HOW_MUCH_RELIGION
    ## Asian.or.Asian.American -0.3721498 NA Asian.or.Asian.American
    ## Mixed.Race               3.8196370 NA              Mixed.Race
    ## Or.some.other.race      -0.1507814 NA      Or.some.other.race
    ## White                    3.1325978 NA                   White
    ## A.man                   -5.6367169 NA                   A.man
    ## In.some.other.way        5.6563851 NA       In.some.other.way

``` r
predVal <- predict(lasso.cv, newx=as.matrix(x))
r2 <- cor(predVal, y)^2
message(paste('R2 =', round(r2,3)))
```

    ## R2 = 0.108

This LASSO regression of the February ;21 data shows very similar
results to those presented in the March ’20 data.

One important difference: 1.6470000 NA F_IDEO

Shows that after controlling for the other variables, each step from “1:
Very conservative” to “5: Very liberal” increases mental distress by
about 1.6% compared to the 2.8% reported previously. Perhaps this
reflects a Democratic victory in the 2020 presidential election.

``` r
F_PARTY_FINAL_name <- names(attr(feb21Dat$F_PARTY_FINAL, "labels"))[match(feb21Dat$F_PARTY_FINAL, attr(feb21Dat$F_PARTY_FINAL, "labels"))]
F_PARTY_IDEO_name <- names(attr(feb21Dat$F_IDEO, "labels"))[match(feb21Dat$F_IDEO, attr(feb21Dat$F_IDEO, "labels"))]
F_PARTY_IDEO_name <- factor(F_PARTY_IDEO_name , levels=names(attr(feb21Dat$F_IDEO, "labels")))

table(F_PARTY_IDEO_name, F_PARTY_FINAL_name)
```

    ##                    F_PARTY_FINAL_name
    ## F_PARTY_IDEO_name   Democrat Independent Refused Republican Something else
    ##   Very conservative       30         109       2        557             66
    ##   Conservative           168         514      17       1465            138
    ##   Moderate              1274        1405      34        579            389
    ##   Liberal               1699         390       6         35             97
    ##   Very liberal           698         136       1         13            140
    ##   Refused                 29          21      27         27             55

## Step 7: Conclusion

Taken as a whole, this analysis shows that people who identify as
‘Liberal’ or ‘Very Liberal’ much more mentally ill / distressed than
they rest of the population. Additionally, after controlling for other
variables, a liberal ideology remains associated with increased mental
distress.
