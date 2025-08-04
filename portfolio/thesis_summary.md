
# Thesis Summary: Predicting Subjective Well-Being (SWB) of European Individuals Using Advanced Machine Learning Techniques

This study predicts happiness levels using data from the [European Social Survey Round 11 (ESS11)](https://www.europeansocialsurvey.org/data/round-index.html), working with 40,156 individuals across Europe. The problem was framed as a binary classification task. Four models—Logistic Regression, Random Forest, LightGBM, and XGBoost—were applied, with **Random Forest** performing best (F1-score: **89%**, recall for the “unhappy” class: **87%**).

## 🔄 Machine Learning Pipeline
<p align="center">
<img src="/assets/portfolio/Eli_Colored_2.png" width="1000">
</p>

The pipeline included:
- Data cleaning and missing value imputation
- Feature encoding and selection
- Stratified train-test splitting (80/20)
- Scaling and handling class imbalance
- Model selection and hyperparameter tuning via 5-fold Random Search

## 📊 Feature Relationships

<p align="center">
<img src="/assets/portfolio/Fig3.png" width="1000">
</p>


A correlation heatmap revealed that **life satisfaction**, **economic confidence**, **trust in police**, and **mental health indicators** (e.g., loneliness, sadness) are strongly associated with happiness.

## 📈 Feature Correlation with Happiness

<p align="center">
<img src="/assets/portfolio/Fig44.png" width="1000">
</p>


These relationships were further confirmed using correlation plots and SHAP values, offering interpretable, data-driven insights that can support policy-making and intervention strategies to enhance societal well-being.
