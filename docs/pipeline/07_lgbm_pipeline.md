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

feature_names:      list[str]   # from FeatureExtractor.get_feature_names()
categorical_cols:   list[str]   # ["sector_code"] — passed to LightGBM
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
| `up5` | `label_up5` | price rises ≥ +5pp within 60min |
| `up3` | `label_up3` | price rises +3~5pp within 60min |
| `sw`  | `label_sw`  | price stays within ±3pp (sideways) |
| `dn3` | `label_dn3` | price drops -3~5pp within 60min |
| `dn5` | `label_dn5` | price drops ≥ -5pp within 60min |

Class weight balancing within LightGBM: `is_unbalance=True` per classifier.

---

## Training Logic

```python
for label in ["up5", "up3", "sw", "dn3", "dn5"]:
    X_train = train[feature_names]
    y_train = train[f"label_{label}"]

    X_val = val[feature_names]
    y_val = val[f"label_{label}"]

    dtrain = lgb.Dataset(X_train, label=y_train,
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

Saved to `models/` directory after each training run.

```
models/
├── {run_id}_up5.lgb
├── {run_id}_up3.lgb
├── {run_id}_sw.lgb
├── {run_id}_dn3.lgb
├── {run_id}_dn5.lgb
└── {run_id}_meta.json     # feature_names, categorical_cols, eval_result, config snapshot
```

`run_id` format: `YYYYMMDD_HHMMSS`

---

## Interface

```python
class LGBMPipeline:
    def __init__(self, config: dict): ...

    def train(
        self,
        train: pd.DataFrame,
        val:   pd.DataFrame,
        feature_names: list[str],
        categorical_cols: list[str],
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

    def save(self, models: dict, run_id: str, eval_result: dict): ...
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

Used in backtest and live inference.
Only `up3` and `up5` predictions trigger a buy signal.

```python
def should_enter(probs: dict, threshold: float = 0.5) -> str | None:
    """
    Returns "up5", "up3", or None.
    up5 takes priority over up3 if both exceed threshold.
    """
    if probs["up5"] >= threshold: return "up5"
    if probs["up3"] >= threshold: return "up3"
    return None
```

Threshold is configurable. Initial value: 0.5.

---

## Constraints

- Five classifiers trained independently — no shared weights or joint loss
- `feature_names` order must match FeatureExtractor.get_feature_names() exactly
- Early stopping patience (50 rounds) is fixed; not exposed to optimizer in phase 1
- Model artifacts must include config snapshot for full reproducibility
- `predict()` returns probabilities, not binary labels — thresholding done by caller
