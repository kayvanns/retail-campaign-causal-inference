# Causal Study of Personalised Campaigns Across Demographic Groups

## Project Motivation
- Randomised controlled trials are the gold standard for causal inference
  but are not always feasible. They are expensive, time-consuming, and
  sometimes operationally impossible in retail settings.
- Vast amounts of observational data exist in modern retail environments.
  Methods that extract causal insight from this data are increasingly
  valuable.
- Combining causal inference with machine learning moves beyond pattern
  recognition and association. It enables estimation of cause-and-effect
  relationships from data that was never designed as an experiment.
- Average treatment effects mask important variation. Households respond
  differently to the same campaign. Identifying who benefits and who does
  not is essential for efficient targeting and resource allocation.

---

## Data
**Source**: Dunnhumby — The Complete Journey  
**Coverage**: 2,500 households, 2 years of transaction data, 30 campaigns  
**Tables used**: transaction_data, campaign_table, campaign_desc, 
coupon_redempt, hh_demographic, product, causal_data

## Research Population

| Decision | Rationale |
|---|---|
| Restricted to households with demographic data | Required for subgroup CATE estimation |
| First campaign exposure only | 45% of household × campaign rows had concurrent overlapping campaigns |
| TypeB and TypeC collapsed into non-personalised | TypeC n=18 after demographic filter (insufficient for stable estimates). Both share uniform coupon distribution |
| 176 households with overlapping first campaigns | Outcome window truncated to day before second campaign began |
| 41 never-treated households excluded | Insufficient for a clean control group |
| **Final sample** | **760 households (512 personalised / 248 non-personalised)** |

---

## Architecture

Medallion lakehouse pipeline built on Databricks Community Edition:

Raw CSV upload (DBFS)

↓

Bronze layer : 8 Delta tables, raw schema preserved

↓

Silver layer : single enriched transaction-level table

joins: transactions, campaigns, demographics, products, features

engineered: treatment flag, window flags, customer_spend, coupon_used

↓

Gold layer: one row per household

pre-campaign confounders, campaign outcomes, demographics

↓

Causal Discovery → Causal Model → Refutation

**All experiment runs tracked with MLflow.**

## Methods
### Causal Discovery
- PC algorithm with Fisher-Z conditional independence tests
- Background knowledge enforced temporal tier ordering:
  demographics → pre-campaign behavior → treatment → outcome
- Three domain knowledge overrides applied to algorithm output
- Recovered DAG used to construct DoWhy CausalModel for refutation
  
### Causal Forest Double Machine Learning
- **Estimator**: CausalForestDML (EconML)
- **Outcome (Y)**: log1p(avg_daily_campaign_spend)
- **Treatment (T)**: TypeA personalised = 1, TypeB/C non-personalised = 0
- **Controls (W)**: pre-campaign behavioral features + technical controls
- **Effect modifiers (X)**: household demographic features
- W variables are partialled out of both Y and T using Gradient Boosting 
  before the causal forest estimates heterogeneous effects as a function of X
  
### Model Refutation

Three refutation tests run via DoWhy on the linear regression baseline estimate:

| Test | Original ATE | New ATE | p-value | Result |
|---|---|---|---|---|
| Placebo treatment | -0.038 | +0.005 | 0.82 | PASS |
| Data subset (80%) | -0.038 | -0.040 | 0.84 | PASS |
| Random common cause | -0.038 | -0.038 | 0.98 | PASS |

---

## Key Findings

### ATE

Two estimators with different assumptions both produce negative, 
statistically insignificant average treatment effects:

| Estimator | ATE | 95% CI | p-value |
|---|---|---|---|
| CausalForestDML | -0.157 | [-0.473, 0.159] | 0.33 |
| Linear regression (DoWhy) | -0.038 | [-0.125, 0.049] | 0.39 |

Personalised campaigns do not produce statistically significant 
spend lift over non-personalised campaigns on average.

### CATE Distribution
<img width="900" height="400" alt="cate_distribution" src="https://github.com/user-attachments/assets/c86b9713-7351-44da-8c5d-ae7cbab4f635" />

The standard deviation of individual treatment effects is 0.162 — 
nearly as large as the ATE itself. Substantial heterogeneity exists 
beneath the null average effect. A meaningful subset of households 
shows near-zero or positive treatment effects.

### Heterogenereity and Feature Importance
<img width="1400" height="400" alt="cate_heterogeneity" src="https://github.com/user-attachments/assets/6822fc6b-c5ed-4a59-a7c8-5408dfffa3b8" />

- Age Group 1 shows the most negative CATE (-0.42)
- Age Group 5 and 6 show the least negative CATE (-0.02 to -0.05)
- Higher income levels consistently show less negative treatment effects
  
<img width="800" height="400" alt="heterogeneity_importance" src="https://github.com/user-attachments/assets/b1e0fdc7-55e4-4c1b-a0f9-1d75459518f0" />

Age and income together explain 65% of why the treatment effect 
varies across households.


## Business Recommendation
- TypeB/C non-personalised campaigns outperform TypeA personalised 
  campaigns on spend across all demographic segments in this study.
- If personalised campaigns are retained, direct them at older, 
  higher-income households where the performance gap with 
  non-personalised campaigns is smallest.
- The volume advantage of TypeB/C (households receive all available 
  coupons rather than 16 targeted ones) likely outweighs the 
  relevance advantage of personalisation at current targeting quality.
- Invest in improving the TypeA targeting mechanism before scaling 
  personalised campaigns further.
  
## Limitations
- **Unconfoundedness**: campaign assignment for TypeA used purchase
  signals not fully captured in available features. Residual confounding
  cannot be ruled out.
- **Obfuscated demographics**: some household classification columns have no
  interpretable labels. Subgroup findings cannot be translated into
  actionable descriptions without the original encoding key.
- **Sample size**: 760 households after eligibility filters. CATE
  estimates for small demographic subgroups carry high variance.
- **Short-term outcome**: daily spend during the campaign window does
  not capture long-term effects on loyalty or lifetime value.
- **No true control group**: all 760 households received a campaign.
  The study estimates the relative effect of campaign type, not the
  absolute effect of receiving any campaign versus none.
  
## Repo Structure
## Reproducibility
