# Module: ClassBalancer

**File:** `src/balancing/balancer.py`

---

## Role

Reduce class imbalance in the labeled feature matrix before model training.
Primary concern: sideways (`label_sw`) class is expected to heavily dominate.
Uses downsampling only — no synthetic sample generation.

Also responsible for session_mode filtering, ensuring that train, val, and test
splits all contain only entry points from the target session. This guarantees
that AUC and winning rate metrics reflect actual model performance within the
session the model will be deployed for.

---

## Input / Output

**Input:**
```python
labeled_df: pd.DataFrame
    # feature matrix joined with label columns
    # columns: [...features..., label_up5, label_up3, label_sw, label_dn3, label_dn5]
    # exactly one label column = 1 per row
    # contains entry points from all sessions (no pre-filtering)
```

**Output:**
```python
balanced_df: pd.DataFrame
    # same schema, reduced row count
    # class distribution within configured ratio bounds
```

---

## Strategy

Each row has exactly one label = 1. Derive `dominant_class` column for grouping:

```python
label_cols = ["label_up5", "label_up3", "label_sw", "label_dn3", "label_dn5"]
df["dominant_class"] = df[label_cols].idxmax(axis=1)
```

**Downsampling logic:**

```
1. Count samples per class
2. Identify minority class (smallest count) → n_min
3. For each class:
     target_n = min(class_count, n_min * max_ratio)
   where max_ratio is configured (default: 3.0)
   → sideways class is capped at n_min * 3.0
4. Random sample target_n from each class (random_state from config)
5. Concatenate and shuffle
```

**Example:**
```
Before:  up5=800, up3=2000, sw=15000, dn3=1800, dn5=600
n_min = 600 (dn5), max_ratio = 3.0 → cap = 1800

After:   up5=800, up3=1800, sw=1800, dn3=1800, dn5=600
         (up5 kept as-is since 800 < 1800)
```

---

## Train / Validation / Test Split

Performed inside `split()`. Session_mode filtering and balancing are both
handled here, in the following order:

```
Split order (inside split()):
1. Apply session_mode filter to full labeled_df:
       "regular"  : keep hour 093000~155900 only
       "pre"      : keep hour 040000~092900 only
       "combined" : keep hour < 160000
       None       : no filter (all sessions retained)
   Filter applied to entire DataFrame before splitting —
   train, val, and test all contain only the target session's entry points.
   This ensures AUC and winning_rate metrics reflect the session the model
   will be deployed for.

2. Random split filtered_df → train_raw / val / test  (before balancing)

3. If balance=True: apply downsampling to train_raw only → train_balanced
   If balance=False: train_raw used as-is → train_balanced = train_raw

4. Return train_balanced, val, test
```

`balance()` is available as a standalone method for reporting and testing purposes.
It must not be called before `split()` in the preprocessing pipeline —
doing so would apply downsampling to the full dataset including val/test.

```python
split_ratios: {train: 0.7, val: 0.15, test: 0.15}  # configurable
random_state: 42                                      # configurable
```

---

## Interface

```python
class ClassBalancer:
    def __init__(self, config: dict): ...

    def balance(
        self,
        labeled_df: pd.DataFrame,
    ) -> pd.DataFrame:
        """
        Downsample to configured class ratio.
        Standalone utility — do not call before split() in pipeline.
        Returns balanced DataFrame.
        """
        ...

    def split(
        self,
        labeled_df: pd.DataFrame,
        balance: bool = True,
        session_mode: str | None = None,
    ) -> tuple[pd.DataFrame, pd.DataFrame, pd.DataFrame]:
        """
        Apply session_mode filter, then split into (train_balanced, val, test).
        session_mode filter applied to full DataFrame before splitting —
        all three splits contain only the target session's entry points.
        Balancing applied to train split only when balance=True.
        session_mode and balance values passed from caller via config.

        Args:
            labeled_df:   full labeled feature matrix (all sessions)
            balance:      whether to apply downsampling to train split
            session_mode: "regular" | "pre" | "combined" | None
                          None = no filter (retain all sessions)
        """
        ...

    def report(self, df: pd.DataFrame) -> dict[str, int]:
        """Return class counts for logging."""
        ...
```

---

## Config Keys (pipeline_config.yaml)

```yaml
class_balancer:
  apply_balance: true     # passed to split() as balance argument
  max_ratio: 3.0          # max multiple of minority class count
  random_state: 42
  split:
    train: 0.70
    val:   0.15
    test:  0.15

entry_detector:
  session_mode: "regular"  # "regular" | "pre" | "combined"
                           # read by caller and passed to split() as session_mode argument
                           # searchable by PipelineOptimizer
```

---

## Constraints

- Session_mode filter applied to full DataFrame before splitting —
  train, val, and test all reflect the target session's distribution
- Balancing applied to train split only — val and test untouched by downsampling
- `split()` is the single entry point in the preprocessing pipeline — `balance()` is not called directly
- `balance` argument to `split()` is read from `config["class_balancer"]["apply_balance"]` by caller
- `session_mode` argument to `split()` is read from `config["entry_detector"]["session_mode"]` by caller
- `session_mode=None` disables filtering — all sessions retained (used for testing)
- No SMOTE or synthetic oversampling
- Random state must be fixed for reproducibility
- `report()` must be called and logged before and after balancing
