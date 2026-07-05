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
Filters applied to train split only — val and test are never filtered.

1. Dead position Case C exclusion (configurable):
       Remove rows where dead_position_case == "C"
       Reason: Case C entries hit the dataset boundary — label is forced label_sw
               regardless of actual price movement. Including them biases the model.
       Config: class_balancer.exclude_case_c: true | false

2. Dead position Case D exclusion (configurable):
       Remove rows where dead_position_case == "D"
       Reason: Case D entries (extended/multi-day halt, exit_price unresolvable)
               are forced to pnl=-1.0 / label_dn5 as a conservative fallback, not
               an observed market outcome — same rationale class as Case C, but
               kept as a separate flag since Case D IS a real (if extreme) pnl
               outcome and some training configurations may prefer to keep it.
       Config: class_balancer.exclude_case_d: false (default — Case D is a
               genuine, if conservative, capital-loss outcome; unlike Case C it
               is not purely a dataset-boundary artifact, so it is retained by
               default and left as an opt-in exclusion)
       Note: Case B (delisting) has no exclusion option — it is always retained,
             consistent with existing treatment prior to this option's addition.

3. Ambiguous sample exclusion (configurable):
       Remove rows where is_ambiguous == True
       Reason: simultaneous bundle-level breach makes the label direction uncertain.
       Config: class_balancer.ambiguous_sample_action: "exclude" | "keep"
               "exclude" → remove from train
               "keep"    → pass to Trainer with reduced weight (sample_weight_col)
```

---

## Mode 1: Temporal Split (split)

Single-pass split into train, val, test by date order.
Replaces random split to prevent future data leakage into training.

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

---

## Mode 2: Rolling Walk-Forward (generate_folds)

Walk-forward fold generation for robust out-of-sample evaluation.
Each fold represents an independent train/val/test window advancing
through time by step_weeks.

Fold parameters can be overridden explicitly by the caller (e.g., PipelineOptimizer
passing different values for outer vs. inner folds). If override parameters are None,
values are read from `config["class_balancer"]`.

```
Fold structure (fixed window):
    train: window_weeks of data ending at fold boundary
    val:   val_weeks immediately after train (with embargo); empty if val_weeks=0
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
    1. Apply session_mode filter (if session_mode is not None)
    2. Slice train/val/test by date ranges
    3. Apply embargo gap (trading days)
    4. Apply pre-balance filters to train
    5. Apply downsampling to train if balance=True
    6. Yield (train_balanced, val, test, fold_meta)

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
    If max_folds is set, stop after that many folds regardless.
```

**Recommended parameters for inner folds (10-month dataset, ~210 trading days):**
```
window_weeks: 8    → ~40 trading days
val_weeks:    2    → ~10 trading days
test_weeks:   2    → ~10 trading days
step_weeks:   2    → ~14–17 folds from 10-month data
embargo_days: 5    → 1 trading week buffer
max_folds:    4    → cap inner folds for nested validation efficiency
```

**Recommended parameters for outer folds (nested validation):**
```
window_weeks: 16   → ~80 trading days (consistent, recent history)
val_weeks:    0    → no val at outer level; val generated separately from outer_train
test_weeks:   6    → ~30 trading days (enough trades for backtest evaluation)
step_weeks:   6    → non-overlapping outer_test windows
embargo_days: 5    → 1 trading week buffer
```

With 10-month (~43 weeks) data, outer fold config yields approximately 3–4 outer folds.

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
          4. Apply pre-balance filters to train (Case C, Case D, ambiguous)
          5. Apply downsampling to train if balance=True

        All three splits contain only target session's entry points.

        Args:
            labeled_df:   full labeled feature matrix (all sessions)
            balance:      whether to apply downsampling to train split
            session_mode: "regular" | "pre" | "combined" | None
        """
        ...

    def report(self, df: pd.DataFrame) -> dict[str, int]:
        """Return class counts per label for logging and diagnostics."""
        ...

    def generate_folds(
        self,
        labeled_df:   pd.DataFrame,
        balance:      bool = True,
        session_mode: str | None = None,
        max_folds:    int | None = None,
        # Explicit fold parameter overrides — if None, read from config["class_balancer"]
        window_weeks: int | None = None,
        val_weeks:    int | None = None,
        test_weeks:   int | None = None,
        step_weeks:   int | None = None,
        embargo_days: int | None = None,
    ) -> Iterator[tuple[pd.DataFrame, pd.DataFrame, pd.DataFrame, dict]]:
        """
        Walk-forward fold generator.
        Yields (train_balanced, val, test, fold_meta) per fold.
        Folds are always yielded in ascending fold_idx order (0, 1, 2, ...).

        If val_weeks=0 (or override val_weeks=0), val is returned as an
        empty DataFrame. Caller is responsible for ignoring it.

        Explicit override parameters take precedence over config values.
        This allows PipelineOptimizer to reuse the same method for both
        inner folds (inner_fold config) and outer folds (outer_fold config)
        without ClassBalancer needing to distinguish between them.

        fold_meta dict keys:
            fold_idx        (int)  : 0-based fold index
            fold_train_end  (str)  : 'YYYYMMDD' last date of train window
            fold_test_start (str)  : 'YYYYMMDD' first date of test window
            fold_test_end   (str)  : 'YYYYMMDD' last date of test window

        Order of operations per fold:
          1. session_mode filter on full df (skipped if session_mode is None)
          2. Compute fold date boundaries using resolved parameters
          3. Slice train/val/test
          4. Apply pre-balance filters to train (Case C, Case D, ambiguous)
          5. Apply downsampling to train if balance=True
          6. Yield (train_balanced, val, test, fold_meta)
        """
        ...
```

---

## Config Keys (pipeline_config.yaml)

```yaml
class_balancer:
  max_ratio: 3.0
  random_state: 42
  apply_balance: true
  exclude_case_c: true
  exclude_case_d: false                # see Pre-Balance Filtering — Case D is a genuine
                                       # (if conservative) outcome, not a boundary artifact
  ambiguous_sample_action: "exclude"   # "exclude" | "keep"
  embargo_days: 5                      # default; overridable per call

  # Temporal split parameters (used by ClassBalancer.split())
  temporal:
    train_pct:    0.70
    val_pct:      0.15
    test_pct:     0.15
    embargo_days: 5    # trading days excluded at train/val and val/test boundaries

  # Inner fold parameters (used by selection/full phases and inner loop of nested validation)
  inner_fold:
    window_weeks: 8
    val_weeks:    2
    test_weeks:   2
    step_weeks:   2
    embargo_days: 5
    max_folds:    4    # cap for nested validation inner loop

  # Outer fold parameters (used by outer loop of nested validation only)
  outer_fold:
    window_weeks: 16
    val_weeks:    0    # no val at outer level; val generated separately from outer_train
    test_weeks:   6
    step_weeks:   6
    embargo_days: 5
```

---

## Constraints

- Pre-balance filters applied to train split only — val and test are never filtered
- session_mode filter applied before split — no cross-session contamination
- Downsampling uses random sampling without replacement (random_state from config)
- `generate_folds()` always yields in ascending fold_idx order — callers may rely on this
- Explicit override parameters in `generate_folds()` take precedence over config values
- `val_weeks=0` results in empty val DataFrame — caller must handle
- `max_folds` limits total fold count; None = no limit
- `session_mode=None` skips session filter — all entry points included (used for outer folds
  where session_mode is a search variable resolved by the inner trial)
- ClassBalancer has no knowledge of "outer" vs. "inner" fold semantics —
  distinction is managed entirely by the caller (PipelineOptimizer)
- `generate_folds()` does NOT accept `holdout_dates` — holdout filtering is the
  caller's responsibility before passing labeled_df
