# Module: DimensionalityReducer

**File:** `src/feature_selection/reducer.py`

---

## Role

Reduce feature dimensionality after initial LightGBM training to combat the curse of dimensionality.
Runs three sequential filters and outputs a reduced feature list for pipeline retraining.

---

## Input / Output

**Input:**
```python
X_train: pd.DataFrame        # feature matrix, training split
X_valid: pd.DataFrame        # feature matrix, validation split
model: lgb.Booster           # trained LightGBM model (any single classifier)
feature_names: list[str]     # column names of X_train
```

**Output:**
```python
selected_features: list[str]   # features surviving all active filters
reduction_report: dict         # counts and removed feature names per stage
```

---

## Three-Stage Pipeline

### Stage 1: Correlation Filter
Remove one feature from each highly correlated pair to eliminate redundancy (e.g., MA(5) stats and MA(10) stats).

```
For each pair (fi, fj) where |corr(fi, fj)| > threshold:
    remove the one with lower LightGBM gain importance
    (or the second one alphabetically if importance not yet available)

Default threshold: 0.95 (configurable)
```

### Stage 2: Importance Filter
Remove features with negligible contribution to model predictions.

```
Compute: gain_importance = model.feature_importance(importance_type="gain")
Remove:  features in the bottom percentile by gain importance

Default: remove bottom 20% (configurable)
```

### Stage 3: SHAP Filter (optional)
Fine-grained removal based on actual prediction contribution.
More expensive — enable after Stage 1+2 have already reduced dimensionality.

```
Compute: mean_abs_shap = |shap_values|.mean(axis=0)
Remove:  features below absolute SHAP threshold

Default: disabled — enable via config when feature count > shap_trigger_threshold
```

---

## Interface

```python
class DimensionalityReducer:
    def correlation_filter(
        self,
        X: pd.DataFrame,
        importance: pd.Series | None = None,
        threshold: float = 0.95,
    ) -> list[str]: ...

    def importance_filter(
        self,
        feature_names: list[str],
        model: lgb.Booster,
        bottom_pct: float = 0.20,
    ) -> list[str]: ...

    def shap_filter(
        self,
        X: pd.DataFrame,
        model: lgb.Booster,
        min_mean_abs_shap: float,
    ) -> list[str]: ...

    def run_all(
        self,
        X_train: pd.DataFrame,
        X_valid: pd.DataFrame,
        model: lgb.Booster,
    ) -> tuple[list[str], dict]:
        """
        Run all enabled stages sequentially.
        Returns (selected_features, reduction_report).
        """
        ...
```

---

## Integration with PipelineOptimizer

```
PipelineOptimizer cycle:

1. Train LightGBM on full feature set  →  baseline AUC
2. Run DimensionalityReducer.run_all() →  selected_features
3. Retrain LightGBM on selected_features → reduced AUC
4. Compare: if reduced_AUC >= baseline_AUC - tolerance → accept reduction
5. Log both results to experiment_log.csv
```

---

## Config Keys (pipeline_config.yaml)

```yaml
dimensionality_reducer:
  correlation_filter:
    enabled: true
    threshold: 0.95
  importance_filter:
    enabled: true
    bottom_pct: 0.20
  shap_filter:
    enabled: false
    min_mean_abs_shap: 0.001
    trigger_threshold: 200    # enable only if feature count exceeds this
  auc_tolerance: 0.005        # acceptable AUC drop from reduction
```

---

## Constraints

- PCA and other transformation-based methods must NOT be used — interpretability of feature importance must be preserved
- Filters operate on feature names only — they do not modify the feature matrix directly
- The caller (PipelineOptimizer) is responsible for subsetting X_train/X_valid after receiving selected_features
- reduction_report must record: input count, output count, removed feature names per stage
