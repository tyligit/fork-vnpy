# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

VeighNa (vnpy) is a Python quantitative trading framework. Version 4.3.0. Primary language is Chinese (Simplified).

## Common Commands

### Setup
```bash
# Ubuntu/Linux
bash install.sh

# macOS
bash install_osx.sh

# Windows
install.bat
```

The install scripts build the TA-Lib C library from source and install `ta-lib==0.6.4` from the project’s private PyPI index (`https://pypi.vnpy.com`).

### Development
```bash
# Install in editable mode with all extras
pip install -e ".[alpha,dev]"

# Linting (required before PR)
ruff check .

# Type checking (required before PR)
mypy vnpy

# Run tests
pytest tests/

# Build packages
uv build
```

### CI
GitHub Actions runs on Windows with Python 3.13. It installs `ta-lib` via `uv pip install ta-lib==0.6.4 --index=https://pypi.vnpy.com`, then runs `ruff check .`, `mypy vnpy`, and `uv build`.

## High-Level Architecture

### Event-Driven Core
`vnpy/event/engine.py` provides `EventEngine`, the central message bus. It distributes `Event` objects (typed by a string `type` field) to registered handlers. A timer thread emits `EVENT_TIMER` every second. All trading activity flows through this engine.

### Trading Platform (`vnpy/trader/`)
`MainEngine` (`engine.py`) coordinates the system. It maintains registries for:
- **Gateways** (`gateway.py`): Connect to broker/exchange APIs. `BaseGateway` is abstract; implementations must be thread-safe, non-blocking, and auto-reconnect. Gateways emit market and trading events.
- **Apps** (`app.py`): Strategy modules (CTA, backtester, etc.). `BaseApp` defines metadata (`app_name`, `engine_class`, `widget_name`).
- **Engines** (`engine.py`): `BaseEngine` subclasses handle specific domains (e.g., CTA strategy engine). Each engine receives references to `MainEngine` and `EventEngine`.

Key data structures live in `object.py` (`TickData`, `BarData`, `OrderData`, `TradeData`, `ContractData`, etc.) and `constant.py` (enums: `Direction`, `Offset`, `Status`, `Exchange`, `Interval`, `Product`).

### Symbol Convention
`vt_symbol` is the universal identifier: `"{symbol}.{exchange.value}"` (e.g., `"rb2505.SHFE"`). Use `utility.extract_vt_symbol()` and `utility.generate_vt_symbol()` to parse/build them.

### Database & Datafeed Adapters
Both are loaded dynamically by name:
- Database: `vnpy_{SETTINGS["database.name"]}` (falls back to `vnpy_sqlite`). See `database.py`.
- Datafeed: `vnpy_{SETTINGS["datafeed.name"]}` (falls back to `vnpy_rqdata`). See `datafeed.py`.

Adapter packages are separate repositories (e.g., `vnpy_ctp`, `vnpy_ctastrategy`, `vnpy_sqlite`). This core repo only defines the abstract base classes.

### Charting (`vnpy/chart/`)
`pyqtgraph`-based K-line charts. `ChartWidget` displays items (`CandleItem`, `VolumeItem`). `BarManager` manages data buffers and aggregation.

### RPC (`vnpy/rpc/`)
ZeroMQ-based cross-process communication. `RpcServer`/`RpcClient` enable distributed deployments.

### Alpha/ML Module (`vnpy/alpha/`) — New in 4.0
A machine-learning quant research pipeline inspired by Microsoft Qlib:
- **`dataset/`**: Feature engineering with Polars. `AlphaDataset` builds training data from Polars expressions. Built-in datasets: `Alpha158`, `Alpha101`.
- **`model/`**: ML model templates. `AlphaModel` is the abstract base. Implementations: `LassoModel`, `LgbModel`, `MlpModel`.
- **`strategy/`**: `AlphaStrategy` template with a `BacktestingEngine` for cross-sectional or time-series strategies.
- **`lab.py`**: `AlphaLab` manages the research workflow: raw bar data → dataset → model → signal → backtest. Stores data as Parquet files in a lab directory.

The alpha module requires the `[alpha]` extras: `polars`, `scipy`, `scikit-learn`, `lightgbm`, `torch`, `pyarrow`, `alphalens-reloaded`.

## Contribution Rules

- Open PRs against the **`dev`** branch, not `master`.
- Run `ruff check .` and `mypy vnpy` before submitting. Both must pass.
- Use type annotations strictly; `mypy` is configured with `disallow_untyped_defs = true`.
