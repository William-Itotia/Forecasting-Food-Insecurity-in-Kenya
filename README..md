# Forecasting Food Insecurity in Kenya

A machine learning pipeline for forecasting food insecurity across Kenyan counties using climate, vegetation, and food pricing data. Built as part of an MSc Data Science thesis at the University of Hertfordshire.

---

## Overview

This script integrates four real-world datasets to build a binary classification model that predicts whether a Kenyan county is in a food crisis (IPC ≥ 3) or not. It covers the full data science pipeline: data ingestion, cleaning, feature engineering, exploratory analysis, and modelling using Random Forest and XGBoost.

---


### Libraries
- pandas, numpy — data manipulation
- geopandas — geospatial mapping
- matplotlib, seaborn — visualisation
- scikit-learn — modelling and evaluation utilities
- xgboost — gradient boosted tree modelling

---

## Data Sources

The following files must be placed in your Google Drive root before running:

| File | Description | Source |
|------|-------------|--------|
| ipcFic_data.csv | IPC food security classifications | FEWS NET |
| ken-ndvi-subnat-full.csv | NDVI vegetation index data | data.humdata.org |
| ken-rainfall-subnat-full.csv | Rainfall data | data.humdata.org |
| ken_admin_boundaries.xlsx | Kenya county admin boundaries | data.humdata.org |
| wfp_food_prices_ken.csv | Maize food prices by county | data.humdata.org  |
| geoBoundaries-KEN-ADM1.geojson | Kenya shapefile for choropleth mapping | geoBoundaries |

---

## Pipeline Overview

### 1. Data Loading and Filtering
Loads all six datasets from Google Drive. IPC data is filtered to Kenya only, current situation classifications, IPC 3.0/3.1 scale, and records from 2019 onwards. Refugee camp entries (Dadaab and Kakuma) are excluded to ensure county-level geographic consistency.

### 2. Data Integration
NDVI and rainfall datasets are merged with a county-level administrative lookup table. All date columns are standardised to monthly frequency. NDVI values are aggregated by mean and rainfall by sum per county per month. IPC duplicates within the same county-month are resolved using mode aggregation, and quarterly assessments are forward-filled to monthly frequency under the assumption that food security conditions change gradually over time.

### 3. Maize Price Imputation
WFP pricing data is filtered to maize only and standardised to KES per kilogram. County name mismatches are resolved (for example, Meru North and Meru South are mapped to Meru). A three-tier imputation strategy achieves 100% price coverage across all 47 counties: first, forward filling within each county using its own historical prices; second, filling remaining gaps with the regional monthly average; and third, applying the national monthly average as a final fallback.

### 4. Exploratory Data Analysis
The following visualisations are generated to understand the data before modelling:
- Choropleth maps showing average IPC, rainfall, NDVI, and maize prices by county across Kenya
- Time series plots for IPC, rainfall, NDVI, and maize prices over the full 2019–2025 period
- Correlation heatmap across key variables
- IPC distribution histogram illustrating class imbalance

### 5. Feature Engineering
The following features are constructed prior to modelling:

| Feature | Description |
|---------|-------------|
| vim_lag1 | Last month's vegetation index — captures autocorrelation in food security conditions |
| vim_6m_avg | 6-month rolling average vegetation — captures longer-term vegetation trends |
| severe_drought | Binary flag indicating both vegetation and rainfall are in the bottom 10th percentile for that county |
| drought_risk | Binary flag combining low rainfall and vegetation below the 25th percentile |
| price_change_1m | Month-on-month percentage change in maize price |
| price_6m_avg | 6-month rolling average maize price |
| price_spike | Binary flag indicating price is more than 20% above its 6-month average |
| price_volatility | 6-month rolling standard deviation of maize price |
| month_sin / month_cos | Circular sine and cosine encoding of month number to capture Kenya's seasonal rainfall patterns |
| ipc_lag1, ipc_lag3, ipc_lag6 | Lagged IPC values at 1, 3, and 6 months to capture autocorrelation and seasonal persistence |
| ipc_deviation | County IPC relative to its own historical average — captures whether conditions are worse than normal for that county |
| rain_x_veg | Interaction term between rainfall and vegetation |
| ipc_lag1_x_drought | Interaction term between lagged IPC and drought risk |

### 6. Train / Test Split
A time-based split is applied rather than random splitting, to respect the temporal order of the data and prevent future information from leaking into the training set. All observations before 2024-01-01 are used for training, and all observations from 2024-01-01 onwards are reserved for testing.

### 7. Class Imbalance Handling
The binary target variable classifies a county-month as a crisis if IPC is 3 or above, and as normal otherwise. Because crisis events represent only around 5% of observations, balanced sample weights are computed to upweight the minority class during training. Crisis weights are further amplified to maximise recall on rare but high-consequence events.

### 8. Modelling
Two models are trained and compared:

**Random Forest Classifier** — an ensemble of 300 decision trees with a maximum depth of 12. Uses the square root of features at each split and is trained with balanced sample weights. Evaluated on precision, recall, F1, and accuracy.

**XGBoost** — a gradient boosted tree model with 200 estimators, a learning rate of 0.05, and a maximum depth of 5. Subsampling and column sampling are applied at each tree to reduce overfitting. Also trained with sample weights and evaluated using MAE, RMSE, R², and crisis-specific recall.

### 9. Evaluation
The primary evaluation metric is recall, chosen because a false negative — predicting a county as food secure when it is actually in crisis — carries a far greater cost than a false positive given the potential loss of lives. Secondary metrics include precision, F1, accuracy, MAE, RMSE, and R². A confusion matrix and full classification report are printed for both models, alongside a feature importance ranking and visualisation of the top 10 predictors.

---

## Key Design Decisions

- **Binary classification over multiclass:** Framing the problem as crisis versus no crisis (IPC ≥ 3) is consistent with established literature and simplifies the class imbalance challenge compared to a five-class IPC prediction task.
- **Recall as primary metric:** Missing a genuine crisis event is far more costly than a false alarm, making recall the most appropriate metric for this humanitarian context.
- **Time-based train/test split:** Random splitting would introduce data leakage by allowing future observations into the training set. A date-based split preserves temporal integrity.
- **Circular month encoding:** Sine and cosine encoding of the month number ensures the model treats December and January as adjacent, correctly capturing Kenya's two rainy seasons (March–May long rains and October–December short rains).
- **County-level analysis:** Refugee camp entries are excluded to maintain consistency at the county administrative level.

---

## Outputs

- Choropleth maps of IPC, rainfall, NDVI, and maize prices across Kenya by county
- Time series trend plots for all key variables across the full study period
- Confusion matrix and classification report for both models
- Feature importance chart showing the top 10 predictors
- Scatter plot of actual versus predicted IPC values

---

