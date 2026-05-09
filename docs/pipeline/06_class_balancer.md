# Module: ClassBalancer

**File:** `src/balancing/balancer.py`

---

## Role

Reduce class imbalance in the labeled feature matrix before model training.
Primary concern: sideways (`label_sw`) class is expected to heavily dominate.
Uses downsampling only — no synthetic sample generation.

---

## Input / Output

**Input:**
```python
labeled_df: pd.DataFrame
    # feature matrix joined with label columns
    # columns: [...features..., label_up5, label_up3, label_sw, label_dn3, label_dn5]
    # exactly one label column = 1 per row
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

Performed inside ClassBalancer after balancing.
Balancing is applied to train split only — validation and test are kept as-is.

```
Split order:
1. Random split full labeled_df → train_raw / val / test  (before balancing)
2. Apply downsampling to train_raw only → train_balanced
3. Return train_balanced, val, test

Rationale: val and test must reflect true class distribution
           to give meaningful AUC and winning rate estimates.
```

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
        """Downsample to configured class ratio. Returns balanced DataFrame."""
        ...

    def split(
        self,
        labeled_df: pd.DataFrame,
    ) -> tuple[pd.DataFrame, pd.DataFrame, pd.DataFrame]:
        """
        Split into (train_balanced, val, test).
        Balancing applied to train only.
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
  max_ratio: 3.0          # max multiple of minority class count
  random_state: 42
  split:
    train: 0.70
    val:   0.15
    test:  0.15
```

---

## Constraints

- Balancing applied to train split only — val and test untouched
- No SMOTE or synthetic oversampling
- Random state must be fixed for reproducibility
- `report()` must be called and logged before and after balancing
