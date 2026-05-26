# Module: ClassBalancer

**File:** `src/balancing/balancer.py`

---

## Role

Reduce class imbalance in the labeled feature matrix before model training.
Primary concern: sideways (`label_sw`) class is expected to heavily dominate.
Uses downsampling only — no synthetic sample generation.

Also responsible for session_mode filtering and temporal splitting,
ensuring that train, val, and test splits reflect the correct session
and time ordering. Two split modes are supported: `temporal` (single split)
and `rolling` (walk-forward fold generation).

---

## Input / Output

**Input:**
```python
labeled_df: pd.DataFrame
    # feature matrix joined with label columns
    # columns: [...features..., label_up5, label_up3, label_sw, label_dn3, label_dn5,
    #           is_dead_position, dead_position_case, is_ambiguous]
    # exactly one label column = 1 per row
    # contains entry points from all sessions (no pre-filtering)
```

**Output (split mode):**
```python
tuple[pd.DataFrame, pd.DataFrame, pd.DataFrame]  # (train_balanced, val, test)
```

**Output (rolling mode):**
```python
Iterator[tuple[pd.DataFrame, pd.DataFrame, pd.DataFrame, dict]]
# yields (train_balanced, val, test, fold_meta) per fold
# fold_meta = {
#   "fold_idx":        int,    # 0-based fold index
#   "fold_train_end":  str,    # 'YYYYMMDD' — last date of train window
#   "fold_test_start": str,    # 'YYYYMMDD' — first date of test window
#   "fold_test_end":   str,    # 'YYYYMMDD' — last date of test window
# }
# Folds are always yielded in ascending fold_idx order (0, 1, 2, ...).
# fold_run_ids[-1] in PipelineOptimizer is guaranteed to correspond to
# the fold with the highest fold_idx.
```

---

## Balancing Strategy

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

## Pre-Balance Filtering

Before balancing is applied to the train split, optional filters are applied
to remove samples that may introduce training bias.

```
Filter 1 — Dead position Case C exclusion (optional):
    if exclude_dead_position_case_c = true:
        remove rows where dead_position_case == "C" from train only
        val and test retain Case C rows (backtest realism)
    Rationale: Case C represents dataset boundary conditions
               (no next-day data), not genuine overnight holds.
               Including them in training introduces a -50% pnl artifact
               that is not representative of real trading outcomes.

Filter 2 — Ambiguous sample exclusion (optional):
    if ambiguous_sample_action = "exclude":
        remove rows where is_ambiguous == True from train only
    if ambiguous_sample_action = "keep" (default):
        retain ambiguous samples with no modification
    val and test always retain ambiguous samples.
    Rationale: ambiguous samples have uncertain labels due to same-bar
               breach. Excluding them from training reduces label noise.

    Note: "weight" handling for ambiguous samples is NOT done here.
    Sample weighting is the responsibility of Trainer, configured
    separately under trainer.use_ambiguous_sample_weight.
    ClassBalancer only decides include/exclude — not weight values.
```

Filter order: Case C exclusion → ambiguous handling → balancing.
All filters applied to train split only. Val and test are never filtered.

---

## Split Modes

### split_method config scope

`split_method` config key applies **only in the PipelineOptimizer context**:
- PipelineOptimizer reads `split_method` to decide whether to call `split()` or `generate_folds()`
- Standalone CLI (`run_preprocess.py`) always calls `split()` regardless of `split_method` config
- `split()` and `generate_folds()` are independently callable methods with no internal
  enforcement of `split_method` — the caller determines which to use

### Mode 1: Temporal Split

Single split based on date ordering. Replaces random split to prevent
future data leakage into training.

```
Split order:
1. Apply session_mode filter to full labeled_df
2. Sort by date ascending
3. Compute split boundaries by date (not by row count):
     unique_dates sorted ascending
     train_dates: first 70% of dates
     val_dates:   next 15% of dates (with embargo gap)
     test_dates:  final 15% of dates (with embargo gap)
4. Apply pre-balance filters to train split
5. Apply downsampling to train split if balance=True
6. Return (train_balanced, val, test)

Embargo gap:
    embargo_days trading days excluded between train/val and val/test boundaries
    prevents information leakage from bars near split boundaries
```

### Mode 2: Rolling Walk-Forward (generate_folds)

Walk-forward fold generation for robust out-of-sample evaluation.
Each fold represents an independent train/val/test window advancing
through time by step_weeks.

```
Fold structure (fixed window):
    train: window_weeks of data ending at fold boundary
    val:   val_weeks immediately after train (with embargo)
    test:  test_weeks immediately after val (with embargo)

Rolling:
    fold 0: train=[week 1 ~ week W],         val=[week W+1 ~ W+V],   test=[week W+V+1 ~ W+V+T]
    fold 1: train=[week 1+S ~ week W+S],     val=[week W+S+1 ~ ...], test=[...]
    fold 2: train=[week 1+2S ~ week W+2S],   ...
    ...
    where W=window_weeks, V=val_weeks, T=test_weeks, S=step_weeks

    Week boundaries are aligned to calendar weeks (Monday start).
    Partial weeks at dataset boundaries are included if >= 3 trading days.

Per fold:
    1. Apply session_mode filter
    2. Slice train/val/test by date ranges
    3. Apply pre-balance filters to train
    4. Apply downsampling to train if balance=True
    5. Yield (train_balanced, val, test, fold_meta)

fold_meta contents:
    fold_idx:        0-based integer index of this fold
    fold_train_end:  last date of train window ('YYYYMMDD')
    fold_test_start: first date of test window ('YYYYMMDD')
    fold_test_end:   last date of test window ('YYYYMMDD')

Yield order:
    Folds are always yielded in ascending fold_idx order (0, 1, 2, ...).
    This guarantee is relied upon by PipelineOptimizer:
        fold_run_ids[-1] always corresponds to MAX(fold_idx) for that trial.

Termination:
    Stop when remaining data after train end is insufficient
    to fill val + test windows (accounting for embargo).
```

**Recommended parameters for 10-month dataset (~210 trading days):**
```
window_weeks: 10   → ~50 trading days, ~12,500 raw train samples
val_weeks:    2    → ~10 trading days, ~2,500 samples
test_weeks:   2    → ~10 trading days, ~2,500 samples
step_weeks:   2    → ~14 folds from 10-month data
embargo_days: 5    → 1 trading week buffer (trading days)
```

These yield ~14 folds covering ~65% of the dataset as out-of-sample.
As data accumulates, fold count increases automatically with no config change.

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
        Temporal split into (train_balanced, val, test).

        Always callable regardless of split_method config.
        split_method config is only enforced by PipelineOptimizer —
        not by this method itself.

        Order of operations:
          1. session_mode filter
          2. Sort by date, split by date boundaries
          3. Apply embargo gap (trading days)
          4. Apply pre-balance filters to train (Case C, ambiguous)
          5. Apply downsampling to train if balance=True

        All three splits contain only target session's entry points.

        Args:
            labeled_df:   full labeled feature matrix (all sessions)
            balance:      whether to apply downsampling to train split
            session_mode: "regular" | "pre" | "combined" | None
        """
        ...

    def generate_folds(
        self,
        labeled_df: pd.DataFrame,
        balance: bool = True,
        session_mode: str | None = None,
    ) -> Iterator[tuple[pd.DataFrame, pd.DataFrame, pd.DataFrame, dict]]:
        """
        Walk-forward fold generator.
        Yields (train_balanced, val, test, fold_meta) per fold.
        Folds are always yielded in ascending fold_idx order (0, 1, 2, ...).

        fold_meta dict keys:
            fold_idx        (int)  : 0-based fold index
            fold_train_end  (str)  : 'YYYYMMDD' last date of train window
            fold_test_start (str)  : 'YYYYMMDD' first date of test window
            fold_test_end   (str)  : 'YYYYMMDD' last date of test window

        Order of operations per fold:
          1. session_mode filter on full df
          2. Compute fold date boundaries (window/val/test/step/embargo)
          3. Slice train/val/test
          4. Apply pre-balance filters to train (Case C, ambiguous)
          5. Apply downsampling to train if balance=True
          6. Yield (train_balanced, val, test, fold_meta)

        Args:
            labeled_df:   full labeled feature matrix (all sessions)
            balance:      whether to apply downsampling to train split
            session_mode: "regular" | "pre" | "combined" | None
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
  apply_balance: true       # passed to split()/generate_folds() as balance argument
  max_ratio: 3.0            # max multiple of minority class count
  random_state: 42

  split_method: "rolling"   # "temporal" | "rolling"
                            # Applies only in PipelineOptimizer context:
                            #   "temporal" → PipelineOptimizer calls split()
                            #   "rolling"  → PipelineOptimizer calls generate_folds()
                            # Standalone CLI (run_preprocess.py) always uses split()
                            # regardless of this setting.

  # Temporal split parameters (used when PipelineOptimizer uses split_method = "temporal")
  temporal:
    train_pct: 0.70
    val_pct:   0.15
    test_pct:  0.15
    embargo_days: 5         # trading days

  # Rolling fold parameters (used when PipelineOptimizer uses split_method = "rolling")
  rolling:
    window_weeks: 10        # fixed train window size
    val_weeks:    2         # validation window per fold
    test_weeks:   2         # test window per fold
    step_weeks:   2         # fold advance step
    embargo_days: 5         # trading days excluded at split boundaries

  # Pre-balance filters (applied to train split only)
  exclude_dead_position_case_c: true   # exclude dataset-boundary dead positions from train
  ambiguous_sample_action: "keep"      # "keep" | "exclude"
                                       # "keep":    retain ambiguous samples (default)
                                       # "exclude": remove is_ambiguous=True from train
                                       # Note: sample weighting for ambiguous samples
                                       #       is configured separately under trainer.*

entry_detector:
  session_mode: "regular"   # read by caller, passed to split()/generate_folds()
```

---

## Constraints

- Temporal ordering must be preserved: train always precedes val, val always precedes test
- Embargo gap applied at both train/val and val/test boundaries (unit: trading days)
- Pre-balance filters applied to train split only — val and test are never filtered
- Balancing applied to train split only — val and test untouched by downsampling
- `split_method` config is enforced only by PipelineOptimizer — `split()` and `generate_folds()` are always callable independently
- `balance()` is available as a standalone method for reporting and testing — not called directly in pipeline
- `session_mode=None` disables session filtering — all sessions retained (used for testing)
- No SMOTE or synthetic oversampling
- Random state must be fixed for reproducibility
- `report()` must be called and logged before and after balancing
- Week boundaries aligned to calendar weeks (Monday); partial weeks >= 3 trading days are included
- Rolling fold count is determined automatically from data length — not configurable directly
- `generate_folds()` always yields folds in ascending fold_idx order — this is a hard guarantee
- `generate_folds()` yields fold_meta as the fourth element of every tuple —
  callers must unpack all four values: `for train, val, test, fold_meta in balancer.generate_folds(...)`
- `ambiguous_sample_action` controls only include/exclude in ClassBalancer;
  sample weight reduction for ambiguous samples is handled by Trainer
  and configured under `trainer.use_ambiguous_sample_weight`
