# Module: BaseModel

**File:** `src/models/base_model.py`

---

## Role

Abstract base class for model trainers. Defines the common interface that all model
implementations must implement (train, evaluate, predict, save, load, feature_importance).
Provides shared artifact management (JSON metadata + pickle model binary).

---

## Input / Output

**Input:**
```python
train: pd.DataFrame    # balanced training split
val:   pd.DataFrame    # validation split
feature_names: list[str]
categorical_cols: list[str] | None   # None if model does not require categorical handling
```

**Output:**
```python
models: M                # subclass-defined model artifacts (e.g. dict[str, lgb.Booster])
eval_result: dict        # evaluation results
predictions: pd.DataFrame  # probability predictions
```

---

## Interface

```python
from typing import Generic, TypeVar

M = TypeVar("M")  # model artifacts type — subclass-defined
                  # e.g. dict[str, lgb.Booster] for LGBMPipeline
                  #      nn.Module for MLPPipeline (planned)

class BaseModel(ABC, Generic[M]):
    @property
    @abstractmethod
    def model_type(self) -> str: ...

    @property
    @abstractmethod
    def model_version(self) -> str: ...

    def train(
        self,
        train: pd.DataFrame,
        val: pd.DataFrame,
        feature_names: list[str],
        categorical_cols: list[str] | None = None,
    ) -> M:
        """
        Train model and return artifacts.
        Return type M is subclass-defined (e.g. dict[str, lgb.Booster] for LGBMPipeline).
        Delegates to _train_impl(). Do not override train() directly —
        subclasses must implement _train_impl() only.
        Callers must treat the return value as opaque and pass it only to
        evaluate(), predict(), save(), feature_importance() of the same instance.
        """
        ...
    def _train_impl(self, train, val, feature_names, categorical_cols) -> M: ...

    def evaluate(self, models: M, test: pd.DataFrame, feature_names: list[str]) -> dict[str, Any]: ...
    def _evaluate_impl(self, models: M, test: pd.DataFrame, feature_names: list[str]) -> dict[str, Any]: ...

    def predict(self, models: M, X: pd.DataFrame) -> pd.DataFrame: ...
    def _predict_impl(self, models: M, X: pd.DataFrame) -> pd.DataFrame: ...

    def feature_importance(self, models: M, importance_type: str = "gain") -> pd.DataFrame: ...
    def _feature_importance_impl(self, models: M, importance_type: str) -> pd.DataFrame: ...

    def save(
        self,
        models: M,
        run_id: str,
        eval_result: dict,
        feature_names: list[str],
        categorical_cols: list[str] | None = None,
    ) -> None: ...
    def _save_model_impl(self, models: M, path: str) -> None: ...

    def load(self, run_id: str) -> tuple[M, dict]: ...
    def _load_model_impl(self, path: str) -> M: ...

    def list_run_ids(self) -> list[str]:
        """
        Return list of run_ids saved under {model_dir}/{model_type}/.
        Scans subdirectory names under the artifact path constructed from
        config["model"]["model_dir"] and self.model_type.
        Utility method for CLI inspection and debugging.
        Not called by pipeline scripts.
        """
        ...
```

---

## Artifact Management

Artifacts saved to `{model_dir}/{model_type}/{run_id}/`:
- `{run_id}.pkl` — serialized model binary (subclass-defined)
- `{run_id}_meta.json` — metadata: `model_type`, `model_version`, `feature_names`, `categorical_cols`, `eval_result`, config snapshot

The artifact directory is created automatically on `__init__`.

---

## Config Keys (pipeline_config.yaml)

```yaml
model:
  model_dir: "models"
  model_type: "lightgbm"  # registered model type key
```

---

## Constructor

```python
def __init__(self, config: dict) -> None: ...
```

Initializes the base class with config-driven settings. Reads `config["model"]["model_dir"]`
and `config["model"]["model_type"]` to construct the artifact directory path.
Creates the directory automatically on init.

---

## Constraints

- Subclasses must implement `_train_impl`, `_evaluate_impl`, `_predict_impl`,
  `_feature_importance_impl`, `_save_model_impl`, `_load_model_impl`
- Subclasses must NOT override public methods (`train`, `evaluate`, `predict`, etc.) directly —
  all customization goes into the corresponding `_impl` methods
- `model_type` and `model_version` properties must be defined by subclass
- `categorical_cols=None` default allows non-LightGBM models to omit categorical handling
- Constructor accepts a `config` dict (see Config Keys above)
- Model instantiation is handled by `src/models/factory.py` via registry pattern —
  `BaseModel` has no knowledge of its subclasses
- Subclasses must declare their artifacts type via `BaseModel[M]`:
  e.g. `class LGBMPipeline(BaseModel[dict[str, lgb.Booster]])`
- `M` is resolved at subclass declaration; erased to `Any` at factory call site.
  Callers must never pass `models` to a different BaseModel instance.
- `list_run_ids()` scans `{model_dir}/{model_type}/` using `self.model_type` —
  utility method for debugging, not called by pipeline scripts

---

## Factory

`src/models/factory.py` maintains a registry of concrete implementations:

| Config `model.model_type` | Concrete class | M (artifacts type) |
|---|---|---|
| `lightgbm` | `LGBMPipeline` | `dict[str, lgb.Booster]` |
| `mlp` | `MLPPipeline` (planned) | `nn.Module` |

`run_train.py` instantiates models exclusively via `create_model(config) -> BaseModel[Any]`.

Note: `create_model()` returns `BaseModel[Any]` — the concrete artifacts type M is
resolved at subclass declaration and is available when the subclass is used directly
(e.g. in tests). At the factory call site, M is erased to Any; callers must treat
the `models` return value of `train()` as opaque and pass it only to `evaluate()`,
`predict()`, `save()`, `feature_importance()` of the same instance.
