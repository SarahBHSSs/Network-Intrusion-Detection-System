# 🛡️ Network Intrusion Detection System 

Machine learning pipeline that detects and classifies network attacks (DDoS, PortScan, Web Attacks) from raw network-flow data, built on the industry-standard **CICIDS2017** dataset.

## Context and objective

A Network Intrusion Detection System (NIDS) needs to tell benign traffic apart from attacks in real time and, ideally, name the attack type so a security team can react appropriately. This project builds that classifier from raw network-flow captures, tackling the two problems that make this task hard in practice:

1. **Severe class imbalance** : attacks are, by nature, rare events buried in a sea of normal traffic (down to 3 samples for one attack type in the raw data).
2. **False negatives are expensive** : a missed attack is far costlier than a false alarm, which drives every metric and resampling choice in this project.

## Dataset

**CICIDS2017** (Canadian Institute for Cybersecurity), labeled network traffic combining benign activity with common attack scenarios captured over multiple days.

- **Raw data**: ~1.5M labeled flows merged from 4 capture files (Monday benign baseline + Friday DDoS/PortScan + Thursday Infiltration), **86 flow-level features** per record (packet timing, flag counts, byte/packet rates, etc.)
- **Stratified 10% sample** (150K rows) kept for tractable iteration, preserving the original class proportions
- **7 original classes**: BENIGN, DDoS, PortScan, Infiltration, and 3 Web Attack subtypes (Brute Force, XSS, SQL Injection)

## Pipeline

```
Multi-file merge  →  Label cleaning  →  Stratified sampling  →  Type/duplicate cleanup
    →  Infinite-value imputation  →  Standardization & encoding  →  EDA & correlation analysis
    →  Binary classification (Attack vs Benign)  →  Multi-class classification (attack type)
```

## Methodology deep-dive

### Data engineering, not just `dropna()`
Two flow-rate features (`Flow Bytes/s`, `Flow Packets/s`) contained `inf` values when traffic was too fast to measure accurately, a common CICIDS2017 quirk. Rather than dropping those rows, they were **imputed with a domain-aware rule**: the global BENIGN maximum for benign flows, and the **local maximum per attack type** for attack flows, since different attacks have very different traffic-rate distributions. That kind of case-by-case data engineering, not the modeling, is usually what separates a working NIDS from a toy one.

### EDA that shapes the modeling strategy, not just describes the data
- **Class imbalance quantified precisely**: a 40,391× ratio between the most and least frequent class, the number that justifies every resampling decision downstream, not just "imbalance detected."
- **Correlation-driven feature selection**: ranked all 80 numeric features by correlation with the target (`PSH Flag Count` led at 0.4578), then explicitly hunted for **redundant feature pairs** (mutual correlation > 0.9) to avoid feeding the model near-duplicate signals.
- Distribution and outlier analysis (skewness, boxplots) on the top-correlated features informed which transformations to apply before scaling.

### Binary classification (Attack vs. Benign)
`SMOTE` oversampling of attacks + random undersampling of benign traffic, then three models compared head-to-head: **XGBoost**, **Random Forest**, and **Logistic Regression** as a linear baseline, deliberately kept in the comparison to show the gap a non-linear model closes (F1 0.88 → 0.99).

### Multi-class classification (naming the attack)
Two pragmatic data decisions before modeling: dropped `Infiltration` (only 3 raw samples, not enough signal to learn from, and SMOTE on 3 points would fabricate more noise than pattern), and merged the three Web Attack subtypes into one `Web_Attack` class to get a viable sample size. Applied **class-specific SMOTE ratios** (e.g. Web_Attack ×14.4 vs. DDoS ×1.2) rather than a single blanket strategy, then compared Random Forest against XGBoost with 5-fold cross-validation.

### Why F1-score, not accuracy
With BENIGN at 80%+ of the data, a model predicting "benign" every time would already score 80% accuracy while catching zero attacks. **F1-score** was chosen deliberately because it's resilient to that imbalance and, critically for intrusion detection, penalizes the false negatives (missed attacks) that matter most operationally.

## Results

| Task | Best model | F1-score | Accuracy | Precision | Recall |
|---|---|---|---|---|---|
| Binary (Attack vs Benign) | Random Forest | **0.9945** | 0.9956 | 0.9950 | 0.9969 |
| Multi-class (4 classes) | XGBoost | **0.9872** (macro) | - | - | - |

- Binary Logistic Regression baseline: F1 = 0.8798, included specifically to show what the tree ensembles buy over a linear model.
- Top predictive signals across both tasks: `PSH Flag Count`, `Protocol`, `Bwd/Fwd Packet Length`, and TCP flag counts, consistent with how DDoS/PortScan traffic differs structurally from normal flows.
- Multi-class confusion matrix shows near-perfect separation even for the smallest retained class (`Web_Attack`, 592/600 correctly identified).
