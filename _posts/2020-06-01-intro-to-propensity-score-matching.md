---
layout:     post
title:      Introduction to Propensity Score Matching with Python
date:       2020-06-01
summary:    Learn how to implement an econometric technique to deal with observational data.
categories: jekyll pixyll
---

**Propensity Score Matching Matching (PSM)** is an econometric technique that allows you to compare a control group and a treatment group when the two groups were not constructed using random assignment. This blog post will provide a very basic overview of PSM and discuss how to implement it in your data project using Python.

## Background

When we conduct an experiment to find the difference between two groups, we would ideally want to randomly assign them to a control and a treatment group. For example:

1. In clinical trials for medical treatments, doctors would use double-blind randomized trials to determine who receives the actual drug and who receives the placebo.

1. In A/B testing for web applications, random assignment is used to determine whether a user will be able to see and/or interact with a new feature change.

Random assignment ensures that the two groups are systematically similar in every way _**except**_ for the treatment variable that you’ve chosen (_e.g._ whether a patient receives a drug or a placebo). This in turn ensures that any difference you find between the control and treatment group is actually due to the treatment variable.

Problems arise when random assignment is possible -- for reasons that may range from constrained resources to experimental ethics. For example, consider an experiment related to smoking:

* On the one hand, it is ethically impossible for you to randomly assign subjects to a smoking or non-smoking group.

* On the other hand, without some way to guarantee that the control and treatment groups are similar in every way except for smoking habits (_e.g._ similar distribution in age, sex, or economic background), there’s no way to know if any difference between the two groups is due to smoking or some other hidden variable.

Thankfully, PSM offers a relatively straightforward way to generate comparable control and treatment groups when you can’t rely on random assignment.

## How Propensity Score Matching Works

On a high level, Propensity Score Matching works by...

* Taking a dataset that consists of a control and a treatment group (designated by a categorical variable in the dataset), and

* Generating a _**new**_ control group that is comparable to the treatment group, using a logistic regression.

Let's see this in action with an example.
