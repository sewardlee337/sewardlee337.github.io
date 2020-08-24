---
layout:     post
title:      Intro to Propensity Score Matching with R
date:       2020-06-20
summary:    A brief tutorial on what PSM is, how it can be used to analyze observational data, and how to implement it using R.
categories: [statistics]
tags: [statistics, r]
---

<img src = "/assets/images/regine-tholen-44cdCyupgDQ-unsplash.jpg">

**Propensity Score Matching Matching (PSM)** is an econometric technique that allows you to compare a control group and a treatment group when the groups were not constructed using random assignment. This tutorial will provide a basic overview of PSM and demonstrate how to implement it using R.



## Background

When you conduct an experiment to find the difference between two groups, you'd ideally want to construct a control group and a treatment group by randomly assigning test subjects between the two groups. For example:

* In clinical trials for medical treatments, doctors use double-blind randomized trials to determine whether a test participant will receive the actual drug or a placebo.

* In A/B testing for web applications, random assignment is used to determine whether a user can see and/or interact with a new feature change.

Random assignment ensures that the two groups are similar in every way **_except_** for the chosen treatment variable (_e.g._ whether a patient receives a drug or a placebo). This in turn ensures that any difference found between the control and treatment group is actually driven by the treatment variable.

Problems arise when random assignment is impossible -- for reasons that may range from constrained resources to experimental ethics. For example, consider a study related to smoking:

* On the one hand, it is ethically impossible for you to randomly assign subjects to a smoking or non-smoking group.

* On the other hand, without some way to guarantee that the control and treatment groups are similar in every way except for smoking habits (_e.g._ similar distribution in age, sex, or economic background), there’s no way to know if any difference between the two groups is due to smoking or some other hidden variable.

Thankfully, PSM offers a relatively straightforward way to generate comparable control and treatment groups when you can’t rely on random assignment.

## How Propensity Score Matching Works

On a high level, Propensity Score Matching works by:

* Taking an observational dataset that consists of a control and treatment group (designated by a categorical variable in the dataset), and

* Generating a **_new_** control group that is comparable to the treatment group.

The steps to generate the new control group are:

1. Train a [logistic regression](https://en.wikipedia.org/wiki/Logistic_regression?wprov=srpw1_0) to predict whether an observation is in the treatment group, using the other independent variables in your dataset (the [covariates](https://support.minitab.com/en-us/minitab/18/help-and-how-to/modeling-statistics/anova/supporting-topics/anova-models/understanding-covariates/)) as the features for your regression model.

2. For each observation in the dataset, use the trained logistic regression to predict whether that observation will belong in the treatment group. The prediction is a probability between 0 and 1. This is your **"propensity score"**.

    > Propensity Score
    >  = P(treatment = 1 | X<sub>1</sub>, X<sub>2</sub>, X<sub>3</sub>, ... X<sub>i</sub>)

3. Once a propensity score has been assigned to every observation, for each observation in the treatment group, find a corresponding observation in the control group with the closest propensity score.

4. Your **_new_** control group will be comprised of observations in your **_original_** control group that have been matched to an observation in your treatment group.

## An Applied Example with R: Classroom Reading

For this applied example, we will use observational data from a paper on whether intervention strategies change student reading behavior.[^1] Each row corresponds to an observed student. The variables in this dataset are:

| Variables   	| Descriptions                                                                   	|
|-------------	|--------------------------------------------------------------------------------	|
| CARS        	| Content Area Reading Strategies Program. 0 = No CARS; 1 = CARS                  |
| SES         	| 0 = No Free Lunch; 1 = Free Lunch                                              	|
| MIN         	| 0 = White; 1 = Non-White                                                       	|
| Gr          	| 1 = 9th Grade; 2 = 10th Grade; 3 = 11th Grade; 4 = 12th Grade                  	|
| FCAT        	| Florida Comprehensive Assessment Test. 1 = Little Success; 5 = Highest Success 	|
| Gen         	| 0 = Female; 1 = Male                                                           	|
| GPA         	| Grade point average on 4.0 scale                                               	|
| Pre 	        | Number of books red per month prior to instructional                           	|
| Post 	        | Number of books red per month after to instructional                           	|

<br>

The ultimate goal of the study was to see whether a reading program (`CARS`) could cause a change in number of books read per month (`Post` - `Pre`). But since random assignment was not used to determine whether a student would receive the additional reading instruction, we will use Propensity Score Matching to create comparable control and treatment groups.

First, let's load the data from this study into our R workspace.

```r
# Run this code to build example dataset

CARS  <- c(0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,1,1,1,1,1,1,1,1,1,1,1,1,1,1)
SES   <- c(1,0,1,1,0,0,0,1,1,0,1,0,1,0,0,1,0,0,1,0,0,0,0,0,0,0,0,1,0,0)
Min   <- c(0,1,0,0,0,0,1,0,0,0,0,0,0,0,1,0,1,1,0,0,0,0,0,1,0,0,0,0,0,1)
Gr    <- c(3,4,3,3,3,1,1,4,2,1,1,3,3,2,1,1,2,4,3,4,4,4,1,4,2,2,1,1,1,2)
FCAT  <- c(4,5,3,5,3,1,4,2,2,2,2,1,2,4,3,2,2,4,4,1,1,2,1,2,2,1,5,3,2,5)
Gen   <- c(0,1,0,0,1,0,1,0,0,1,0,1,0,0,0,0,1,0,0,1,0,0,1,1,0,1,1,0,1,1)
GPA   <- c(2.9,3.1,2.85,2.75,3.25,3.45,3.5,3.35,3.6,3.6,3.75,3.21,2.75, 2.95,3.45,3.05,3.85,3.75,3.25,3.33,3.05,3.25,3.35,3.85,4,2.9,3.45,3.1,3.25,3.75)
Pre   <- c(4,3,4,2,1,3,3,1,3,3,1,2,1,4,3,2,1,4,3,1,1,2,1,4,2,1,2,2,2,2)
Post  <- c(4,2,4,3,1,3,2,1,4,3,1,2,2,4,3,2,2,4,2,3,3,3,1,4,3,2,2,2,3,3)

raw_data <- data.frame(CARS, SES, Min, Gr, FCAT, Gen, GPA, Pre, Post)
```

Our raw dataset contains 16 rows of data for which `CARS` = 0 (representing students who did not receive reading intervention) and 14 rows of data for which `CARS` = 1 (representing students who received reading intervention). Since data in the two groups are not comparable, we will use PSM to create a new dataset with comparable control and treatment groups.

Performing PSM in R is easy with help from the [**MatchIt**](https://cran.r-project.org/web/packages/MatchIt/index.html) package. Install the package if you haven't done so already. Then, import it:

```r
# Install the package if you haven't already

install.packages('MatchIt')

# Import the package to R workspace

library(MatchIt)
```
The main function we'll rely upon is `matchit()`, which takes the following as its arguments:
* A formula that specifies your treatment variable and the other independent variables (the covariates), in the form of `treat ~ x1 + x2`
* Your data
* The method used to perform matching

In our case, since data is held in the dataframe `raw_data` and `CARS` is the treatment variable, we implement PSM by first running:

```r
# PSM Part 1: Create MatchIt object

m.out <- matchit(CARS ~ SES + Min + Gr + FCAT + Gen + GPA + Pre,
                 data = raw_data, method = 'nearest')
```

The code above creates the **MatchIt** output object `m.out`. The second step is to pass `m.out` into the function `match.data()`.

```r
# PSM Part 2: Generate dataset with matched control and
# treatment groups

m.data <- match.data(m.out)
```

In the code above, the function `match.data()` outputs the dataframe `m.data`. `m.data` is our new dataset with matched groups. It contains 14 rows for which `CARS` = 0, and 14 rows for which `CARS` = 1. We can now compare data from the control and treatment group as though they were created in a controlled setting.

## Limitations

Propensity Score Matching is a very useful technique, but it has a major methodological limitation. Your ability to account for selection bias and create truly comparable groups is limited by what covariates you have on-hand to generate propensity scores. If there's a variable omitted from your dataset that can cause your control and treatment groups to systematically differ, PSM won't be able to help you.

## Additional Resources

What is Propensity Score Matching:
* [YouTube: "An intuitive introduction to Propensity Score Matching"](https://www.youtube.com/watch?v=ACVyPp1Fy6Y)
* [YouTube: "Propensity Score Matching - A Quick Introduction"](https://www.youtube.com/watch?v=KlL2EVLnLX8)

How to use the **MatchIt** library:
* [CRAN Reference Manual](https://cran.r-project.org/web/packages/MatchIt/MatchIt.pdf)
* [Stuart, Elizabeth & King, Gary & Imai, Kosuke & Ho, Daniel. (2011). MatchIT: Nonparametric Preprocessing for Parametric Causal Inference. Journal of Statistical Software. 42. 10.18637/jss.v042.i08. ](https://imai.fas.harvard.edu/research/files/matchit.pdf)


---
[^1]: [Lane, Forrest & To, Yen & Shelley, Kyna & Henson, Robin. (2012). An Illustrative Example of Propensity Score Matching with Education Research. Career and Technical Education Research. 37. 187-212. 10.5328/cter37.3.187.](https://www.researchgate.net/publication/273061804_An_Illustrative_Example_of_Propensity_Score_Matching_with_Education_Research)
