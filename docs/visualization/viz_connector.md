# Module: VizConnector

**File:** `src/visualization/viz_connector.py`

---

## Role

Abstract base class that defines the visualization interface for the auto-scalping system.
All future visualization modules (trade charts, equity curves, feature importance plots, etc.)
must implement this interface.

This provides a stable contract so that pipeline scripts (`run_preprocess.py`,
`run_train.py`, `run_backtest.py`) can swap visualization backends without
changing pipeline code.

---

## Design Principles

- **Abstract only** — no concrete rendering logic in this file
- **Backend-agnostic** — supports matplotlib, plotly, altair, or any future library
- **Pipeline-friendly** — all methods accept pandas/numpy types, not file paths
- **Extensible** — new visualization types added as new abstract methods

---

## Interface

```python
class VizConnector(ABC):

    # ── Lifecycle ──────────────────────────────────────────────────────

    def __init__(self, config: dict): ...
        """Initialize with pipeline config subset.

        Args:
            config: Dict containing at minimum a 'visualization' key
                    with backend-specific parameters.

        Config keys:
            visualization.output_dir: str | None
                Directory to save rendered figures. If None, figures
                are returned in-memory only (for testing).
            visualization.backend: str
                Backend identifier (e.g. "matplotlib", "plotly").
        """

    def save(self, name: str) -> str | None: ...
        """Save the current figure to disk.

        Args:
            name: Base name for the saved figure file.

        Returns:
            Absolute file path if saved, None if output_dir is None.
        """

    # ── Trade Visualization ───────────────────────────────────────────

    @abstractmethod
    def plot_equity_curve(self, trade_log: pd.DataFrame) -> Any: ...
        """Render an equity curve from trade log data.

        Args:
            trade_log: DataFrame as returned by BacktestEngine._build_trade_log().

        Returns:
            Opaque figure handle (backend-dependent).
        """

    @abstractmethod
    def plot_pnl_distribution(self, trade_log: pd.DataFrame) -> Any: ...
        """Render a histogram of P&L percentages.

        Args:
            trade_log: DataFrame as returned by BacktestEngine._build_trade_log().
        """

    # ── Prediction Visualization ──────────────────────────────────────

    @abstractmethod
    def plot_prediction_timeline(
        self,
        predictions: pd.DataFrame,
        ohlcv: pd.DataFrame,
    ) -> Any: ...
        """Overlay model predictions on price chart.

        Args:
            predictions: DataFrame with prediction probabilities and labels.
            ohlcv: OHLCV DataFrame for the same ticker/date range.
        """

    # ── Model Visualization ───────────────────────────────────────────

    @abstractmethod
    def plot_feature_importance(
        self,
        feature_names: list[str],
        importances: np.ndarray,
    ) -> Any: ...
        """Render a horizontal bar chart of feature importances.

        Args:
            feature_names: List of feature names in importance order.
            importances: Array of importance values (same length).
        """

    # ── Cleanup ───────────────────────────────────────────────────────

    def close(self) -> None:
        """Release any backend resources (figures, sessions, etc.)."""
        ...
```

---

## Usage Pattern

```python
# Pipeline script creates the connector via factory
connector = create_connector(config)

# Use it
figure = connector.plot_equity_curve(trade_log)
connector.save("equity_curve")

# Clean up
connector.close()
```

---

## Factory Pattern

A module-level factory function creates the appropriate concrete implementation:

```python
def create_connector(config: dict) -> VizConnector: ...
```

Supported backends (registered via a registry pattern):

| Backend key | Concrete class |
|---|---|
| `matplotlib` | `MatplotlibVizConnector` (default) |
| `plotly` | `PlotlyVizConnector` (future) |

---

## Constraints

- The abstract base class itself must have no third-party dependencies
  beyond `pandas`, `numpy`, and `typing`
- Concrete backend implementations live in submodules
  (e.g. `src/visualization/backends/matplotlib.py`)
- All abstract methods MUST be documented with args and return types
- Default methods (like `close`) may use lazy imports

---

## Config Keys

```yaml
visualization:
  output_dir: "data/visualizations"   # null = in-memory only
  backend: "matplotlib"                # backend identifier
```
