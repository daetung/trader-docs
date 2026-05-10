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
categorical_cols: list[str] | None
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

    def train(self, train, val, feature_names, categorical_cols=None) -> Any: ...
    def _train_impl(self, train, val, feature_names, categorical_cols) -> Any: ...

    def evaluate(self, models, test, feature_names) -> dict[str, Any]: ...
    def _evaluate_impl(self, models, test, feature_names) -> dict[str, Any]: ...

    def predict(self, models, X) -> pd.DataFrame: ...
    def _predict_impl(self, models, X) -> pd.DataFrame: ...

    def feature_importance(self, models, importance_type="gain") -> pd.DataFrame: ...
    def _feature_importance_impl(self, models, importance_type) -> pd.DataFrame: ...

    def save(self, models, run_id, eval_result, feature_names, categorical_cols) -> None: ...
    def _save_model_impl(self, models, path) -> None: ...

    def load(self, run_id) -> tuple[Any, dict]: ...
    def _load_model_impl(self, path) -> Any: ...

    def list_models(self) -> list[str]: ...
```

---

## Artifact Management

Artifacts saved to `{model_dir}/{model_type}/{run_id}/`:
- `{run_id}.pkl` — serialized model binary (subclass-defined)
- `{run_id}_meta.json` — metadata (model_type, model_version, feature_names, config, eval_result)

---

## Config Keys (pipeline_config.yaml)

```yaml
model:
  model_dir: "models"
  model_type: "lightgbm"  # registered model type key
```

## Constructor

```python
def __init__(self, config: dict) -> None: ...
```

Initializes the base class with config-driven settings. Creates the model artifact
directory automatically on init.

## Constraints

- Subclasses must implement `_train_impl`, `_evaluate_impl`, `_predict_impl`,
  `_feature_importance_impl`, `_save_model_impl`, `_load_model_impl`
- `model_type` and `model_version` properties must be defined
- Constructor accepts a `config` dict (see Config Keys above)
- Model instantiation is handled by `src/models/factory.py` via registry pattern —
  `BaseModel` has no knowledge of its subclasses

## Factory

`src/models/factory.py` maintains a registry of concrete implementations:

| Config `model.type` | Concrete class |
|---|---|
| `lightgbm` | `LGBMPipeline` |
| `mlp` | `MLPPipeline` (planned) |

`run_train.py` instantiates models exclusively via `create_model(config) -> BaseModel`.
