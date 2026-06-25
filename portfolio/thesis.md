---
layout: page
title: ""
subtitle: "What Makes Europeans Happy? A Machine Learning Study"
tags: [Machine Learning, Subjective Well-Being, Happiness Prediction, European Social Survey, Data Science, Random Forest, SHAP, Classification, Policy Impact]
---

<h4>Predicting Subjective Well-Being (SWB) of European Individuals Using Advanced Machine Learning Techniques</h4>

<p align='justify'>
This study predicts happiness levels using data from <a href="https://ess.sikt.no/en/?tab=overview" target="_blank">European Social Survey Round 11 (ESS11)</a>, working with 40,156 individuals across Europe. The problem was framed as a binary classification task. Four models—Logistic Regression, Random Forest, LightGBM, and XGBoost—were applied, with <strong>Random Forest</strong> performing best (F1-score: <strong>89%</strong>, recall for the "unhappy" class: <strong>87%</strong>).
</p>

<h4>Machine Learning Pipeline</h4>

<p align="center">
  <img src="/assets/portfolio/Eli Colored 2.png" alt="Pipeline Diagram" width="600">
</p>

The pipeline included:
- Data cleaning and missing value imputation
- Feature encoding and selection
- Stratified train-test splitting (80/20)
- Scaling and handling class imbalance
- Model selection and hyperparameter tuning via 5-fold Random Search

<h4>Feature Relationships</h4>

<p align="center">
  <img src="/assets/portfolio/Fig3.png" alt="Correlation Heatmap" width="800">
</p>

A correlation heatmap revealed that **life satisfaction**, **economic confidence**, **trust in police**, and **mental health indicators** (e.g., loneliness, sadness) are strongly associated with happiness.

<h4>Feature Correlation with Happiness</h4>

<p align="center">
  <img src="/assets/portfolio/Fig44.png" alt="Feature Correlation Plot" width="800">
</p>

These relationships were further confirmed using correlation plots and SHAP values, offering interpretable, data-driven insights that can support policy-making and intervention strategies to enhance societal well-being.
