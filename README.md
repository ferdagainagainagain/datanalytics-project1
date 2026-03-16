# Assignment 07 — Network Intrusion Detection (Security Breach)

**Course:** Data Analytics, AY 2025-26
**Professor:** Fabio Crestani & Dr. Ana-Maria Bucur
**Students:** Ferdinando Giordano & Gianluca Viviano

---

## Objective

Build a supervised classification model that distinguishes **normal** network connections from **attack** connections using the KDD Cup 1999 dataset.

The dataset contains 4 attack categories: **DoS**, **Probe**, **R2L**, and **U2R** — making this a **5-class problem** (including normal traffic).

---

## Project Structure

```
project1/
├── 07_SecurityBreach.ipynb     # Main notebook (all code + analysis)
├── guidelines/                 # Assignment PDFs
│   ├── 07_SecurityBreach (2).pdf
│   └── 20-da-projects.pdf
├── datasets/                   # ⚠️ NOT in repo — see setup below
│   ├── TrainingData            # 494,021 rows
│   └── TestData                # 311,029 rows
├── class_distribution.png      # EDA output
├── categorical_features.png    # EDA output
├── correlation_heatmap.png     # EDA output
├── feature_distributions.png   # EDA output
├── outlier_rates.png           # EDA output
├── pca_variance.png            # PCA explained variance curve
├── pca_2d.png                  # PCA 2D cluster projection
├── confusion_matrices.png      # Model evaluation
└── per_class_f1.png            # Per-class F1 comparison
```

---

## Setup

### 1. Clone the repo

```bash
git clone <repo-url>
cd project1
```

### 2. Create and activate the virtual environment

```bash
python3 -m venv analyticsenv
source analyticsenv/bin/activate      # macOS / Linux
# or
analyticsenv\Scripts\activate         # Windows
```

### 3. Install dependencies

```bash
pip install pandas numpy matplotlib seaborn scikit-learn
```

### 4. Download the dataset

The datasets are excluded from the repo (too large). Download the **KDD Cup 1999** dataset from the UCI ML Repository:

> https://kdd.ics.uci.edu/databases/kddcup99/kddcup99.html

- Download `kddcup.data.gz` → rename to `TrainingData`, place in `datasets/`
- Download `corrected.gz` → rename to `TestData`, place in `datasets/`

The files should have **no header row** — the notebook assigns column names automatically.

### 5. Run the notebook

```bash
jupyter notebook 07_SecurityBreach.ipynb
```

Run all cells in order (Kernel → Restart & Run All).

---

## Notebook Walkthrough

| Section | What it does |
|---|---|
| **0. Imports** | Libraries and global seed for reproducibility |
| **1. Data Loading** | Loads raw KDD files, strips label suffixes, maps ~40 attack subtypes to 5 high-level categories |
| **2. EDA** | Class distribution, categorical feature analysis, numerical distributions, correlation heatmap, IQR outlier rates |
| **3. Preprocessing** | Label encoding for 3 categorical features, StandardScaler for PCA, PCA explained-variance curve + 2D visualization |
| **4. Modeling** | Trains Random Forest (200 trees) and Histogram Gradient Boosting (200 iterations) |
| **5. Evaluation** | Accuracy, macro F1, weighted F1, per-class classification reports, normalized confusion matrices, feature importance |

---

## Results

### Overall Performance (Test Set)

| Model | Accuracy | Macro F1 | Weighted F1 |
|---|---|---|---|
| Random Forest | 92.3% | 0.584 | 0.905 |
| Hist. Gradient Boosting | 92.5% | **0.591** | **0.907** |

> **Macro F1 is the primary metric** — raw accuracy is misleading on this imbalanced dataset (DoS alone is 79% of samples).

### Per-Class F1

| Class | Random Forest | Hist. Gradient Boosting |
|---|---|---|
| dos | 0.985 | 0.985 |
| normal | 0.837 | 0.845 |
| probe | 0.820 | 0.794 |
| **r2l** | **0.048** | **0.065** |
| **u2r** | **0.230** | **0.264** |

### Key Observations

- **DoS** is detected near-perfectly — its traffic patterns (extreme byte counts, high error rates) are highly distinctive.
- **R2L and U2R** are the hard cases. These attacks look superficially like normal connections at the packet level, and U2R has only ~52 training examples (<0.01% of data).
- **PCA** shows clear separation of DoS and Normal clusters; R2L/U2R overlap significantly with normal traffic in 2D projection.
- **Feature importance** highlights `src_bytes`, `dst_bytes`, connection flags, and host-level statistics as the primary discriminators.
- **Gradient Boosting** marginally outperforms Random Forest, especially on rare classes — consistent with boosting's stronger bias toward correcting minority-class errors.

---

## Known Limitations & What Needs to Be Improved

These are the concrete next steps for whoever continues this work:

### High Priority

1. **Switch to a modern dataset**
   KDD Cup 1999 is synthetic data from a 1999 DARPA simulation. It is widely criticized for being non-representative of real traffic. Consider: **NSL-KDD**, **CICIDS-2017**, **UNSW-NB15**, or **CIC-IDS-2018**.

2. **Fix R2L detection (F1 = 0.05)**
   R2L is practically undetected. Options to explore:
   - SMOTE / ADASYN oversampling for minority classes
   - Cost-sensitive learning (adjust `class_weight` per class, not just `balanced`)
   - Separate binary classifier for R2L vs. rest
   - Anomaly detection approach for rare attack types

3. **Hyperparameter tuning**
   No cross-validation or grid/random search was performed. Both models use defaults or rough guesses. Use `RandomizedSearchCV` or Optuna to find better `n_estimators`, `max_depth`, `learning_rate`, etc.

### Medium Priority

4. **Handle redundant features**
   Multiple feature pairs have Pearson r > 0.99 (e.g., the entire `serror_rate` family). Removing redundant features could speed up training and potentially improve generalization. Try Variance Inflation Factor (VIF) analysis or recursive feature elimination.

5. **Statistical comparison between models**
   The difference in macro F1 (0.584 vs 0.591) may not be statistically significant. Add a McNemar test or a bootstrapped confidence interval to verify.

6. **Add a proper validation split**
   Currently the model is trained on the full `TrainingData` and evaluated on `TestData` with no held-out validation set during development. Add a stratified train/val/test split or k-fold cross-validation.

### Lower Priority (Nice to Have)

7. **Try deep learning for sequential features**
   Network connections have temporal structure. An LSTM or Transformer-based model on connection sequences could improve detection of behavioral attack patterns that single-connection classifiers miss.

8. **Add SHAP explanations**
   Replace MDI feature importance (which can be biased for high-cardinality features) with SHAP values for model-agnostic, per-prediction explainability.

9. **Explore two-stage classification**
   Stage 1: binary normal vs. attack. Stage 2: multi-class among attack types. This architecture often performs better when class sizes are highly asymmetric.

10. **Benchmark against published results**
    Compare results to papers using the same dataset/split to contextualize performance.

---

## References

- Leskovec, J., Rajaraman, A., & Ullman, J. D. *Mining of Massive Datasets*, Ch. 11 (PCA) and Ch. 12 (Classification).
- KDD Cup 1999 dataset: https://kdd.ics.uci.edu/databases/kddcup99/kddcup99.html
