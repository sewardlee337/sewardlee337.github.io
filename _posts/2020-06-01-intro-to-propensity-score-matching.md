---
layout:     post
title:      Intro to Propensity Score Matching with Python
date:       2020-06-01
summary:    Learn how to use PSM to analyze observational data, and how to implement it from scratch using Python.
categories: statistics python
---

**Propensity Score Matching Matching (PSM)** is an econometric technique that allows you to compare a control group against treatment group when the groups were not constructed using random assignment. This post will provide a basic overview of PSM and discuss how to implement it using Python.

## Background

When we conduct an experiment to find the difference between two groups, we would ideally want to randomly assign them to a control and a treatment group. For example:

* In clinical trials for medical treatments, doctors would use double-blind randomized trials to determine who receives the actual drug and who receives the placebo.

* In A/B testing for web applications, random assignment is used to determine whether a user will be able to see and/or interact with a new feature change.

Random assignment ensures that the two groups are systematically similar in every way **_except_** for the treatment variable that you’ve chosen (_e.g._ whether a patient receives a drug or a placebo). This in turn ensures that any difference you find between the control and treatment group is actually due to the treatment variable.

Problems arise when random assignment is impossible -- for reasons that may range from constrained resources to experimental ethics. For example, consider an experiment related to smoking:

* On the one hand, it is ethically impossible for you to randomly assign subjects to a smoking or non-smoking group.

* On the other hand, without some way to guarantee that the control and treatment groups are similar in every way except for smoking habits (_e.g._ similar distribution in age, sex, or economic background), there’s no way to know if any difference between the two groups is due to smoking or some other hidden variable.

Thankfully, PSM offers a relatively straightforward way to generate comparable control and treatment groups when you can’t rely on random assignment.

## How Propensity Score Matching Works

On a high level, Propensity Score Matching works by:

* Taking an observational dataset that consists of a control and treatment group (designated by a categorical variable in the dataset), and

* Generating a **_new_** control group that is comparable to the treatment group.

The steps to generate the new control group are:

1. Train a [logistic regression](https://en.wikipedia.org/wiki/Logistic_regression?wprov=srpw1_0) to predict whether an observation is in the treatment group, with the other independent variables in your dataset as your features.

2. For each observation in the dataset, use the trained logistic regression to predict whether that observation will belong in the treatment group. The prediction is a probability between 0 and 1. This is your **"propensity score"**.

> Propensity Score = P(treatment = 1 | X<sub>1</sub>, X<sub>2</sub>, X<sub>3</sub>, ... X<sub>i</sub>)

3. Once each observation has a propensity score, for each observation in the treatment group, find the corresponding observation in the control group with the closest propensity score. _(Note: Some observations in the control group may be matched upon multiple times. Treat repeated matches as separate observations.)_

4. Your **_new_** control group will be comprised of observations in your **_original_** control group that have been matched to an observation in your treatment group.

## An Example with Python: Student Reading Habits

Let's consider an example related to whether additional classroom instruction will change student reading habits.[^1] Some students were subject to the Content Area Reading Strategies Program ("CARS") while others were not, but random assignment wasn't used to determine which students received this additional instruction.

The following code sets up our environment and builds the example dataset.

```Python
# Import data analysis packages

import numpy as np
import pandas as pd

# Run this code to build the data for our example

data = pd.DataFrame({
    'CARS': [0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,1,1,1,1,1,1,1,1,1,1,1,1,1,1],
    'SES' : [1,0,1,1,0,0,0,1,1,0,1,0,1,0,0,1,0,0,1,0,0,0,0,0,0,0,0,1,0,0],
    'Min' : [0,1,0,0,0,0,1,0,0,0,0,0,0,0,1,0,1,1,0,0,0,0,0,1,0,0,0,0,0,1],
    'Gr'  : [3,4,3,3,3,1,1,4,2,1,1,3,3,2,1,1,2,4,3,4,4,4,1,4,2,2,1,1,1,2],
    'FCAT': [4,5,3,5,3,1,4,2,2,2,2,1,2,4,3,2,2,4,4,1,1,2,1,2,2,1,5,3,2,5],
    'Gen' : [0,1,0,0,1,0,1,0,0,1,0,1,0,0,0,0,1,0,0,1,0,0,1,1,0,1,1,0,1,1],
    'GPA' : [2.9,3.1,2.85,2.75,3.25,3.45,3.5,3.35,3.6,3.6,3.75,3.21,2.75,
             2.95,3.45,3.05,3.85,3.75,3.25,3.33,3.05,3.25,3.35,3.85,4,2.9,
             3.45,3.1,3.25,3.75],
    'Pre' : [4,3,4,2,1,3,3,1,3,3,1,2,1,4,3,2,1,4,3,1,1,2,1,4,2,1,2,2,2,2],
    'Post': [4,2,4,3,1,3,2,1,4,3,1,2,2,4,3,2,2,4,2,3,3,3,1,4,3,2,2,2,3,3]
})
```
This observational dataset comes from a paper relating to education research reading, where:

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

The ultimate goal of the study is to see whether a reading program (`CARS`) can drive a change in number of books read per month (`Post` - `Pre`). But since random assignment was not used to determine whether a student would receive the additional reading instruction, we will need to use Propensity Score Matching to create comparable control and treatment groups.

```python
# Import from sklearn.

from sklearn.linear_model import LogisticRegression

# Separate treatment from  other variables.

X = data[['SES', 'Min', 'Gr', 'FCAT', 'Gen', 'GPA', 'Pre']]
y = data['CARS']

# Fit a logistic regression.

model = LogisticRegression(random_state=0).fit(X,y)

# Predict probability that each observation should be in treatment group.
# These are your propensity scores.

y_prob = model.predict_proba(X)
propensity = pd.DataFrame(y_prob)[1]

data['propensity'] = propensity
```


## A Warning

## Further resources

* https://www.youtube.com/watch?v=ACVyPp1Fy6Y



---
[^1]: Lane, Forrest & To, Yen & Shelley, Kyna & Henson, Robin. (2012). An Illustrative Example of Propensity Score Matching with Education Research. Career and Technical Education Research. 37. 187-212. 10.5328/cter37.3.187.
