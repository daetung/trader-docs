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
models: Any              # implementation-defined model artifacts
eval_result: dict        # evaluation results
predictions: pd.DataFrame  # probability predictions
```

---

## Interface

```python
class BaseModel(ABC):
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
    ) -> Any: ...
    def _train_impl(self, train, val, feature_names, categorical_cols) -> Any: ...

    def evaluate(self, models, test, feature_names) -> dict[str, Any]: ...
    def _evaluate_impl(self, models, test, feature_names) -> dict[str, Any]: ...

    def predict(self, models, X) -> pd.DataFrame: ...
    def _predict_impl(self, models, X) -> pd.DataFrame: ...

    def feature_importance(self, models, importance_type="gain") -> pd.DataFrame: ...
    def _feature_importance_impl(self, models, importance_type) -> pd.DataFrame: ...

    def save(
        self,
        models,
        run_id: str,
        eval_result: dict,
        feature_names: list[str],
        categorical_cols: list[str] | None = None,
    ) -> None: ...
    def _save_model_impl(self, models, path) -> None: ...

    def load(self, run_id) -> tuple[Any, dict]: ...
    def _load_model_impl(self, path) -> Any: ...

    def list_models(self) -> list[str]: ...
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
- `model_type` and `model_version` properties must be defined by subclass
- `categorical_cols=None` default allows non-LightGBM models to omit categorical handling
- Constructor accepts a `config` dict (see Config Keys above)
- Model instantiation is handled by `src/models/factory.py` via registry pattern —
  `BaseModel` has no knowledge of its subclasses

---

## Factory

`src/models/factory.py` maintains a registry of concrete implementations:

| Config `model.model_type` | Concrete class |
|---|---|
| `lightgbm` | `LGBMPipeline` |
| `mlp` | `MLPPipeline` (planned) |

`run_train.py` instantiates models exclusively via `create_model(config) -> BaseModel`.
