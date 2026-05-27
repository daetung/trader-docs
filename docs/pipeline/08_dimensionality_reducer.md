# Module: DimensionalityReducer

**File:** `src/feature_selection/reducer.py`

---

## Role

Reduce feature dimensionality after initial model training to combat the curse of dimensionality.
Runs three sequential filters and outputs a reduced feature list for pipeline retraining.

Model-agnostic by design: all model-dependent operations (importance calculation,
SHAP computation) are delegated to an `ImportanceProvider` implementation.
Swapping the underlying model (LightGBM → MLP) requires only a provider
substitution — no changes to `DimensionalityReducer` itself.

---

## Input / Output

**Input:**
```python
X_train: pd.DataFrame          # feature matrix, training split
X_valid: pd.DataFrame          # feature matrix, validation split
provider: ImportanceProvider   # model-specific importance/SHAP provider
feature_names: list[str]       # column names of X_train
```

**Output:**
```python
selected_features: list[str]   # features surviving all active filters
reduction_report: dict         # counts, removed feature names per stage,
                               # importance_scores, and selected_features list
```

---

## ImportanceProvider Interface

```python
class ImportanceProvider(ABC):
    """
    Abstract interface for model-dependent importance and SHAP computation.
    One concrete implementation per model type.
    All implementations must be instantiated after model training is complete.
    get_importance() is guaranteed to return meaningful scores at call time.
    """

    @abstractmethod
    def get_importance(self, feature_names: list[str]) -> pd.Series:
        """
        Return feature importance scores indexed by feature name.
        For multi-classifier models (e.g. LightGBM 5-class), averaging
        across classifiers is handled here.
        Averaging strategy controlled by:
            config["dimensionality_reducer"]["importance_averaging"]
            "uniform"        — simple mean across classifiers
            "sample_weighted" — weighted by training sample count per classifier

        Returns:
            pd.Series: index=feature_name, values=importance score (float)
        """
        ...

    @abstractmethod
    def get_shap_values(
        self,
        X: pd.DataFrame,
    ) -> np.ndarray:
        """
        Return mean absolute SHAP values per feature.
        Shape: (n_features,) — averaged over samples and classifiers.
        Averaging strategy same as get_importance().

        When shap_subsample_n is set in config, X is pre-subsampled
        by the caller (DimensionalityReducer.shap_filter) before
        being passed to get_shap_values(). The provider itself does
        not apply subsampling — it operates on whatever X is passed.

        Returns:
            np.ndarray: shape (n_features,), dtype float
        """
        ...
```

---

## Concrete Providers

### LGBMImportanceProvider

```python
class LGBMImportanceProvider(ImportanceProvider):
    def __init__(
        self,
        models: dict[str, lgb.Booster],    # {"up5", "up3", "sw", "dn3", "dn5"}
        train_labels: dict[str, pd.Series] | None = None,
                                            # required for sample_weighted averaging
                                            # key = label name, value = y_train Series
        config: dict,
    ): ...
```

**`get_importance()` logic:**
```
For each classifier in models:
    imp[label] = booster.feature_importance(importance_type="gain")

if importance_averaging == "uniform":
    result = mean(imp.values())

if importance_averaging == "sample_weighted":
    weight[label] = train_labels[label].shape[0]   # n_samples per classifier
    result = weighted_mean(imp.values(), weights=weight.values())
```

**`get_shap_values()` logic:**
```
For each classifier in models:
    explainer = shap.TreeExplainer(booster)
    shap_vals[label] = |explainer.shap_values(X)|.mean(axis=0)

averaging: same strategy as get_importance()

Note: X passed here is already subsampled if shap_subsample_n is configured.
      Provider does not apply additional subsampling.
```

### MLPImportanceProvider (deferred — interface defined for forward compatibility)

```python
class MLPImportanceProvider(ImportanceProvider):
    def __init__(
        self,
        model: nn.Module,
        X_ref: pd.DataFrame,      # reference dataset for permutation importance
        y_ref: pd.Series,
        config: dict,
    ): ...
```

**`get_importance()` logic:**
```
sklearn.inspection.permutation_importance(model, X_ref, y_ref, n_repeats=10)
→ result.importances_mean as pd.Series
```

**`get_shap_values()` logic:**
```
explainer = shap.GradientExplainer(model, X_ref)
→ |explainer.shap_values(X)|.mean(axis=0)
```

---

## Provider Factory

```python
def create_importance_provider(
    model_type: str,
    models: Any,
    train_labels: dict[str, pd.Series] | None,
    config: dict,
) -> ImportanceProvider:
    """
    Factory for ImportanceProvider instances.
    model_type: "lightgbm" | "mlp"
    models: dict[str, lgb.Booster] for LightGBM; nn.Module for MLP
    train_labels: required when importance_averaging == "sample_weighted"
    Raises ValueError if train_labels is None and averaging == "sample_weighted".
    """
    ...
```

**Caller example (run_train.py):**
```python
provider = create_importance_provider(
    model_type   = config["model"]["model_type"],
    models       = trained_models,
    train_labels = {
        label: train_df[f"label_{label}"]
        for label in ["up5", "up3", "sw", "dn3", "dn5"]
    },
    config = config,
)
reducer = DimensionalityReducer(config)
selected_features, report = reducer.run_all(X_train, X_valid, provider)
```

---

## Three-Stage Pipeline

### Stage 1: Correlation Filter

Operates on X_train data only (no model training required at this stage).
Importance scores are sourced from the already-trained provider passed to `run_all()`.
`get_importance()` is called once in `run_all()` before any filter stage and
reused across all stages — not called again inside individual filter methods.

```
importance_scores = provider.get_importance(feature_names)   # called once in run_all()

For each pair (fi, fj) where |corr(fi, fj)| > threshold:
    remove the feature with lower importance_scores value
    (alphabetical tiebreak if scores are equal)

Default threshold: 0.95 (configurable)
```

### Stage 2: Importance Filter

Uses the same `importance_scores` obtained in `run_all()` — no additional provider call.

```
Remove features in the bottom percentile by importance score.

Default: remove bottom 20% (configurable)
```

### Stage 3: SHAP Filter (optional)

Delegates to provider for SHAP computation.
Supports subsampling to control computation time when the full training
set is large relative to the number of features.

```
if shap_subsample_n is not None AND len(X_train) > shap_subsample_n:
    X_shap = X_train.sample(n=shap_subsample_n, random_state=config_random_state)
else:
    X_shap = X_train

shap_vals = provider.get_shap_values(X_shap)
Remove features below absolute SHAP threshold.

Subsampling rationale:
    SHAP computation scales with O(samples × features × trees).
    For selection phase with ~6,875 train samples and ~500 features,
    subsampling to 500 rows reduces SHAP time by ~14× with minimal
    impact on feature rank ordering for selection purposes.
    Full sample SHAP is recommended for final model analysis.

Default: disabled
Enabled automatically when feature count > shap_trigger_threshold (configurable)
```

---

## reduction_report Structure

```python
reduction_report: dict = {
    "importance_scores":      pd.Series,   # feature → importance score
                                           # used across correlation and importance filters
    "selected_features":      list[str],   # feature names surviving all active filters
                                           # mirrors run_all() first return value
                                           # used by frequency_vote() in PipelineOptimizer
    "input_feature_count":    int,
    "output_feature_count":   int,
    "stages": {
        "correlation_filter": {
            "removed_count":    int,
            "removed_features": list[str],
        },
        "importance_filter": {
            "removed_count":    int,
            "removed_features": list[str],
        },
        "shap_filter": {
            "enabled":          bool,
            "removed_count":    int,
            "removed_features": list[str],
        },
    },
}
```

`importance_scores` is included in `reduction_report` to enable:
- Audit trail of which importance values drove feature selection decisions
- PipelineOptimizer to include per-fold importance in `feature_selection_log.json`
- Post-hoc analysis of feature selection stability across folds

`selected_features` is included in `reduction_report` to enable:
- `frequency_vote()` to aggregate results across folds without a separate accumulation list
- Consistent serialization of per-fold selection results in `feature_selection_log.json`

---

## Interface

```python
class DimensionalityReducer:
    def __init__(self, config: dict): ...

    def correlation_filter(
        self,
        X: pd.DataFrame,
        importance_scores: pd.Series,
        threshold: float = 0.95,
    ) -> list[str]:
        """
        importance_scores: pre-computed from provider.get_importance() in run_all().
        Not called internally — caller (run_all) passes scores directly.
        """
        ...

    def importance_filter(
        self,
        feature_names: list[str],
        importance_scores: pd.Series,
        bottom_pct: float = 0.20,
    ) -> list[str]:
        """
        importance_scores: same pre-computed scores from run_all().
        """
        ...

    def shap_filter(
        self,
        X: pd.DataFrame,
        provider: ImportanceProvider,
        min_mean_abs_shap: float,
    ) -> list[str]:
        """
        Apply SHAP-based feature filter.
        Subsamples X to shap_subsample_n rows before passing to provider
        if shap_subsample_n is set and len(X) > shap_subsample_n.
        Provider receives the (possibly subsampled) X directly.
        """
        ...

    def run_all(
        self,
        X_train: pd.DataFrame,
        X_valid: pd.DataFrame,
        provider: ImportanceProvider,
    ) -> tuple[list[str], dict]:
        """
        Run all enabled stages sequentially.

        Calls provider.get_importance() exactly once at entry.
        Passes resulting importance_scores to correlation_filter()
        and importance_filter() directly — no additional provider calls
        for importance within this method.

        Returns (selected_features, reduction_report).
        reduction_report includes importance_scores, selected_features,
        and per-stage removal info for audit/logging.
        selected_features is also stored in reduction_report["selected_features"]
        for cross-fold aggregation via frequency_vote().
        """
        ...
```

---

## frequency_vote()

```python
def frequency_vote(
    reduction_reports: list[dict],
    threshold: float,
) -> list[str]:
    """
    Aggregate per-fold selected_features lists via frequency voting.
    Used by PipelineOptimizer selection phase after all folds complete.

    A feature is included in the result if it appears in the selected_features
    of at least `threshold` fraction of folds.

    Args:
        reduction_reports: list of reduction_report dicts, one per fold.
                           Each dict must contain "selected_features": list[str].
        threshold:         minimum fraction of folds in which a feature must appear
                           to be retained (e.g. 0.5 = majority vote).
                           Read from config["dimensionality_reducer"]["vote_threshold"].

    Logic:
        feature_counts = Counter of appearances across all folds' selected_features
        total_folds    = len(reduction_reports)
        result         = [f for f, cnt in feature_counts.items()
                          if cnt / total_folds >= threshold]
        result sorted by descending frequency, then alphabetical tiebreak.

    Returns:
        list[str] — feature names surviving the vote threshold.
        Empty list if no features survive (raises warning).

    Note: called only from PipelineOptimizer selection phase.
          Not a general-purpose utility — input schema is tightly coupled
          to reduction_report["selected_features"] from run_all().
    """
    ...
```

---

## Integration with PipelineOptimizer

```
Selection phase (PipelineOptimizer):

For each fold in generate_folds():
    1. Train model on full feature set              → fold model
    2. create_importance_provider(...)              → provider
    3. reducer.run_all(X_train, X_valid, provider)  → selected_features, reduction_report
       (reduction_report["selected_features"] populated automatically)
    4. Accumulate reduction_reports list
    5. Include reduction_report["importance_scores"] in feature_selection_log

After all folds:
    6. frequency_vote(reduction_reports, threshold=vote_threshold)
       → confirmed selected_features
    7. Save to configs/selected_features.json
    8. Save feature_selection_log.json (includes per-fold importance_scores)
    (No per-fold retraining on selected features)

Exploitation phase (PipelineOptimizer):
    DimensionalityReducer not used — selected_features loaded from JSON
```

---

## Config Keys (pipeline_config.yaml)

```yaml
dimensionality_reducer:
  importance_averaging: "uniform"   # "uniform" | "sample_weighted"
                                    # searchable by PipelineOptimizer
  vote_threshold: 0.5               # fraction of folds a feature must appear in
                                    # to be included in selected_features
                                    # used by PipelineOptimizer frequency_vote()
                                    # range: (0.0, 1.0]; 0.5 = majority vote
  correlation_filter:
    enabled: true
    threshold: 0.95
  importance_filter:
    enabled: true
    bottom_pct: 0.20
  shap_filter:
    enabled: false
    min_mean_abs_shap: 0.001
    trigger_threshold: 200          # enable only if feature count exceeds this
    shap_subsample_n: 500           # max samples passed to SHAP computation
                                    # null = use full X_train (no subsampling)
                                    # recommended: 500 for selection phase speed
  auc_tolerance: 0.005              # acceptable AUC drop from reduction
                                    # used by run_train.py --reduce CLI flow only
```

---

## Constraints

- `DimensionalityReducer` must have zero direct imports of `lightgbm`, `torch`,
  or any model library — all model access via `ImportanceProvider` only
- PCA and other transformation-based methods must NOT be used —
  feature interpretability must be preserved
- Filters operate on feature names only — they do not modify the feature matrix directly
- The caller is responsible for subsetting X_train/X_valid after receiving `selected_features`
- `run_all()` calls `provider.get_importance()` exactly once; result passed to all filter stages
- `correlation_filter()` and `importance_filter()` receive `importance_scores` as a parameter — not the provider
- `reduction_report` must include `importance_scores`, `selected_features`, input/output counts,
  and removed features per stage
- `train_labels` is required when `importance_averaging == "sample_weighted"`;
  `create_importance_provider()` must raise `ValueError` if not provided in that case
- `MLPImportanceProvider` implementation is deferred to MLP phase;
  its interface is defined here for forward compatibility
- SHAP subsampling applied in `shap_filter()` before provider call —
  provider receives subsampled X and does not apply additional subsampling
- `shap_subsample_n: null` disables subsampling; full X_train passed to provider
- Random state for SHAP subsampling sourced from `config["class_balancer"]["random_state"]`
  for reproducibility consistency across pipeline
- `vote_threshold` read from `config["dimensionality_reducer"]["vote_threshold"]`
  by PipelineOptimizer; not used by DimensionalityReducer itself
- `frequency_vote()` called only from PipelineOptimizer selection phase —
  input schema is tightly coupled to reduction_report["selected_features"]
