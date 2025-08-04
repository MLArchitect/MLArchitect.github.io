---
layout: page
title: "Predicting Subjective Well-Being (SWB) of European Individuals Using Advanced Machine Learning Techniques"
cover-img: /assets/portfolio/Fig3.png
tags: [Machine Learning, Subjective Well-Being, Happiness Prediction, European Social Survey, Data Science, Random Forest, SHAP, Classification, Policy Impact]

---


<p align='justify'>

This study predicts happiness levels using data from <a href="https://ess.sikt.no/en/?tab=overview" target="_blank">European Social Survey Round 11 (ESS11)</a>, working with 40,156 individuals across Europe. The problem was framed as a binary classification task. Four models—Logistic Regression, Random Forest, LightGBM, and XGBoost—were applied, with **Random Forest** performing best (F1-score: **89%**, recall for the “unhappy” class: **87%**).

 🔄 Machine Learning Pipeline

<p align="center">
  <img src="/assets/portfolio/Eli Colored 2.png" alt="Pipeline Diagram" width="600">
</p>

The pipeline included:
- Data cleaning and missing value imputation
- Feature encoding and selection
- Stratified train-test splitting (80/20)
- Scaling and handling class imbalance
- Model selection and hyperparameter tuning via 5-fold Random Search

 📊 Feature Relationships

<p align="center">
  <img src="/assets/portfolio/Fig3.png" alt="Correlation Heatmap" width="800">
</p>

A correlation heatmap revealed that **life satisfaction**, **economic confidence**, **trust in police**, and **mental health indicators** (e.g., loneliness, sadness) are strongly associated with happiness.

 📈 Feature Correlation with Happiness

<p align="center
