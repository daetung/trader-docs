# Module: LightGBM Pipeline

**File:** `src/models/lgbm_pipeline.py`

---

## Role

Train 5 independent binary classifiers (one per label class) using LightGBM.
Evaluate per-class AUC on validation and test sets.
Save and load model artifacts for reuse in backtest and inference.

---

## Input / Output

**Input:**
```python
train: pd.DataFrame    # balanced train split (from ClassBalancer)
val:   pd.DataFrame    # unbalanced validation split
test:  pd.DataFrame    # unbalanced test split
# all splits contain [...features..., label_up5, label_up3, label_sw, label_dn3, label_dn5]

feature_names:      list[str]         # from FeatureExtractor.get_feature_names()
categorical_cols:   list[str] | None  # ["sector_code", "day_of_week", "halt_reason_code"]
                                      # derived from FeatureExtractor.get_feature_schema()
                                      # None if not applicable
sample_weight_col:  str | None        # column name for per-sample weights; None = uniform weights
```

**Output:**
```python
models: dict[str, lgb.Booster]
    # keys: "up5", "up3", "sw", "dn3", "dn5"

eval_result: dict
    # {
    #   "up5":  {"val_auc": 0.72, "test_auc": 0.71},
    #   "up3":  {"val_auc": 0.68, "test_auc": 0.67},
    #   ...
    #   "mean_val_auc":  0.69,
    #   "mean_test_auc": 0.68,
    # }
```

---

## Classifier Structure

Five fully independent binary classifiers.
Each uses its own target label column:

| Classifier key | Target column | Positive class meaning |
|---|---|---|
| `up5` | `label_up5` | price rises ≥ +5pp within holding period |
| `up3` | `label_up3` | price rises +3~5pp within holding period |
| `sw`  | `label_sw`  | price stays within ±3pp (sideways) |
| `dn3` | `label_dn3` | price drops -3~5pp within holding period |
| `dn5` | `label_dn5` | price drops ≥ -5pp within holding period |

Class weight balancing within LightGBM: `is_unbalance=True` per classifier.

---

## Sample Weights

Per-sample weights can be passed to LightGBM training via `sample_weight_col`.
This allows downstream callers (e.g. PipelineOptimizer, Trainer) to reduce
the influence of ambiguous or uncertain samples without fully excluding them.

```
If sample_weight_col is not None:
    weights = train[sample_weight_col].values
    dtrain = lgb.Dataset(..., weight=weights)
Else:
    dtrain = lgb.Dataset(...)  # uniform weights

Val and test sets always use uniform weights regardless of sample_weight_col.
```

**Interaction between `is_unbalance=True` and `sample_weight_col`:**
```
LightGBM applies is_unbalance by computing a class-level weight from the
positive/negative ratio, then multiplying it into each sample's gradient.

When sample_weight_col is also provided, LightGBM multiplies the class-level
weight (from is_unbalance) with the per-sample weight. The effective weight
for each sample is:

    effective_weight = class_weight(is_unbalance) × per_sample_weight

For ambiguous samples with per_sample_weight=0.5, this means their
influence is halved relative to non-ambiguous samples of the same class.
The intended behavior (reduced influence of ambiguous samples) is achieved,
though the absolute scale depends on is_unbalance's class ratio calculation.

ClassBalancer already applies downsampling before training, partially
addressing class imbalance. is_unbalance=True provides additional correction
for residual imbalance after downsampling (max_ratio=3.0 cap leaves some
imbalance). Both mechanisms are intentionally combined.
```

**Intended use case — ambiguous samples:**
```
is_ambiguous=True rows may be assigned a reduced weight (e.g. 0.5)
by the caller before passing to Trainer, rather than being excluded.
This preserves sample count while reducing label noise influence.

Weight column generation is the caller's responsibility (Trainer or Optimizer).
LGBMPipeline only reads the column if provided — no weight logic here.
```

`sample_weight_col` is `None` by default. When not provided, all samples
receive equal weight (standard LightGBM behavior with is_unbalance correction).

---

## Training Logic

```python
for label in ["up5", "up3", "sw", "dn3", "dn5"]:
    X_train = train[feature_names]
    y_train = train[f"label_{label}"]
    weights = train[sample_weight_col].values if sample_weight_col else None

    X_val = val[feature_names]
    y_val = val[f"label_{label}"]

    dtrain = lgb.Dataset(X_train, label=y_train,
                         weight=weights,
                         categorical_feature=categorical_cols)
    dval   = lgb.Dataset(X_val,   label=y_val,
                         categorical_feature=categorical_cols,
                         reference=dtrain)

    model = lgb.train(
        params,
        dtrain,
        valid_sets=[dval],
        callbacks=[lgb.early_stopping(50), lgb.log_evaluation(100)]
    )
    models[label] = model
```

---

## Default Parameters

All from `pipeline_config.yaml`. These are starting values — tuned by PipelineOptimizer.

```yaml
lgbm_params:
  objective: binary
  metric: auc
  boosting_type: gbdt
  num_leaves: 31
  max_depth: 6
  min_data_in_leaf: 100
  feature_fraction: 0.7
  bagging_fraction: 0.8
  bagging_freq: 1
  lambda_l1: 0.1
  lambda_l2: 0.1
  learning_rate: 0.05
  n_estimators: 1000       # max; early stopping applies
  is_unbalance: true
  verbose: -1
```

---

## Model Artifacts

Saved to `{model_dir}/{model_type}/{run_id}/` directory after each training run.
`model_type` for this class is `"lightgbm"`. `model_dir` is read from `config["model"]["model_dir"]`.

```
models/lightgbm/{run_id}/
├── {run_id}_up5.lgb
├── {run_id}_up3.lgb
├── {run_id}_sw.lgb
├── {run_id}_dn3.lgb
├── {run_id}_dn5.lgb
└── {run_id}_meta.json     # feature_names, categorical_cols, eval_result, config snapshot
```

`run_id` format: `YYYYMMDD_HHMMSS` (generated via `utils.generate_run_id()`)

---

## Interface

```python
class LGBMPipeline(BaseModel[dict[str, lgb.Booster]]):
    @property
    def model_type(self) -> str:
        return "lightgbm"

    @property
    def model_version(self) -> str: ...

    def train(
        self,
        train: pd.DataFrame,
        val:   pd.DataFrame,
        feature_names: list[str],
        categorical_cols: list[str] | None = None,
        sample_weight_col: str | None = None,
    ) -> dict[str, lgb.Booster]: ...

    def evaluate(
        self,
        models: dict[str, lgb.Booster],
        test: pd.DataFrame,
        feature_names: list[str],
    ) -> dict: ...

    def predict(
        self,
        models: dict[str, lgb.Booster],
        X: pd.DataFrame,
    ) -> pd.DataFrame:
        """
        Returns probability scores per class.
        columns: [prob_up5, prob_up3, prob_sw, prob_dn3, prob_dn5]
        """
        ...

    def save(
        self,
        models: dict[str, lgb.Booster],
        run_id: str,
        eval_result: dict,
        feature_names: list[str],
        categorical_cols: list[str] | None = None,
    ) -> None: ...

    def load(self, run_id: str) -> tuple[dict[str, lgb.Booster], dict]: ...

    def feature_importance(
        self,
        models: dict[str, lgb.Booster],
        importance_type: str = "gain",
    ) -> pd.DataFrame:
        """Per-classifier and averaged feature importance."""
        ...
```

---

## Entry Decision Logic

Entry signal determination is the responsibility of the **caller** (BacktestEngine or live inference),
not the model. `predict()` returns raw probabilities; the caller applies `execution_common.resolve_signal()`
against a configurable threshold.

```python
# Caller-side logic (BacktestEngine, Inferencer)
from execution_common import resolve_signal

probs = model.predict(models, X)   # → prob_up5, prob_up3, prob_dn3, prob_dn5, ...
threshold          = config["backtest"]["entry_threshold"]
suppress_threshold = config["backtest"]["suppress_threshold"]
signal = resolve_signal(row, threshold, suppress_threshold)
# → "up5" | "up3" | None
```

`resolve_signal()` is defined in `src/utils/execution_common.py`. See execution_common.md for full spec.

Threshold and suppress_threshold are configurable via `config["backtest"]`.
Initial values: entry_threshold=0.5, suppress_threshold=0.5.

---

## Constraints

- Five classifiers trained independently — no shared weights or joint loss
- `feature_names` order must match FeatureExtractor.get_feature_names() exactly
- `categorical_cols` derived from FeatureExtractor.get_feature_schema() by caller
- Early stopping patience (50 rounds) is fixed; not exposed to optimizer in phase 1
- Model artifacts must include `feature_names`, `categorical_cols`, and config snapshot for full reproducibility
- `predict()` returns probabilities, not binary labels — thresholding done by caller via `execution_common.resolve_signal()`
- `sample_weight_col` applied to train only — val and test always use uniform weights
- Weight column must exist in train DataFrame if `sample_weight_col` is not None — caller is responsible
- `is_unbalance=True` and `sample_weight_col` may be used simultaneously; effective weight = class_weight × per_sample_weight
- `run_id` generated via `utils.generate_run_id()`
