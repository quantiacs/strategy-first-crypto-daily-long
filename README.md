# Q18 Quick Start

This trading strategy is designed for the [Quantiacs](https://quantiacs.com/contest) platform, which hosts competitions
for trading algorithms. Detailed information about the competitions is available on
the [official Quantiacs website](https://quantiacs.com/contest).

## How to Run the Strategy

### In an Online Environment

The strategy can be executed in an online environment using Jupiter or JupiterLab on
the [Quantiacs personal dashboard](https://quantiacs.com/personalpage/homepage). To do this, clone the template in your
personal account.

### In a Local Environment

To run the strategy locally, you need to install the [Quantiacs Toolbox](https://github.com/quantiacs/toolbox).

## Strategy Overview

This Jupyter notebook shows you the basic steps for taking part to the Q18 NASDAQ-100 Stock Long-Short contest.

Example
-------

```python
import xarray as xr

import qnt.ta as qnta
import qnt.backtester as qnbt
import qnt.data as qndata
import qnt.exposure as qnexp


def load_data(period):
    return qndata.stocks.load_ndx_data(tail=period)


def strategy(data):
    close = data.sel(field="close")
    is_liquid = data.sel(field="is_liquid")  # this field tags NASDAQ-100 stocks
    sma_slow = qnta.sma(close, 200).isel(time=-1)
    sma_fast = qnta.sma(close, 20).isel(time=-1)
    # 1 - long position (**buy**), -1 - short position (**sell**)
    weights = xr.where(sma_slow < sma_fast, 1, -1)
    weights = weights * is_liquid  # trade only NASDAQ-100 stocks
    # Normalize positions and cut big positions
    weights_sum = abs(weights).sum('asset')
    weights = xr.where(weights_sum > 1, weights / weights_sum, weights)
    weights = qnexp.cut_big_positions(weights=weights, max_weight=0.049)
    return weights


weights = qnbt.backtest(
    competition_type="stocks_nasdaq100",
    load_data=load_data,
    lookback_period=365 * 4,
    start_date="2006-01-01",
    strategy=strategy,
    analyze=True,
    build_plots=True,
    check_correlation=False
)
```

Single-pass version
-------------------

```python
import xarray as xr

import qnt.ta as qnta
import qnt.data as qndata
import qnt.output as qnout
import qnt.stats as qns
import qnt.exposure as qnexp

data = qndata.stocks.load_ndx_data(min_date="2005-01-01")

close = data.sel(field="close")
is_liquid = data.sel(field="is_liquid")
sma_slow = qnta.sma(close, 200)
sma_fast = qnta.sma(close, 20)
weights = xr.where(sma_slow < sma_fast, 1, -1)
weights = weights * is_liquid

# Normalize positions and cut big positions
weights_sum = abs(weights).sum('asset')
weights = xr.where(weights_sum > 1, weights / weights_sum, weights)
weights = qnexp.cut_big_positions(weights=weights, max_weight=0.049)

weights = qnout.clean(weights, data, "stocks_nasdaq100")

qnout.check(weights, data, "stocks_nasdaq100", check_correlation=False)

# calc stats
stats = qns.calc_stat(data, weights.sel(time=slice("2006-01-01", None)))
display(stats.to_pandas().tail())

# Graph
performance = stats.to_pandas()["equity"]
import qnt.graph as qngraph

qngraph.make_plot_filled(performance.index, performance, name="PnL (Equity)", type="log")

qnout.write(weights)  # To participate in the competition, save this code in a separate cell.
```
