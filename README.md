# Tiny VaR Engine (Historical vs Gaussian)

This repo is a small market-risk exercise: compute and compare **1-day 95% Value-at-Risk (VaR)** for a single equity/ETF using two standard approaches:
1) **Historical VaR** (empirical quantile of past returns)  
2) **Parametric (Gaussian) VaR** (assume returns are Normal with mean/vol estimated from data)

The script also computes **rolling (time-varying) VaR** and runs a basic **exceptions backtest**.

---

## What the code does (step by step)

### 1) Download prices
- Fetch daily adjusted close prices with `yfinance`.
- Use `auto_adjust=True` so the “Close” series reflects splits/dividends (total-return style pricing).

### 2) Compute daily log-returns
Returns are computed as:

$$
r_t = \ln\left(\frac{P_t}{P_{t-1}}\right)
$$

Log-returns are convenient because they add over time and behave well under continuous-time assumptions.

### 3) Compute 1-day 95% VaR (full sample)
VaR is treated as a **positive loss threshold**.

**Historical VaR**
- Take the 5th percentile (left tail) of returns:

$$
q_{0.05} = \text{Quantile}_{0.05}(r)
$$

- Report VaR as:

$$
\text{VaR}^{\text{hist}}_{0.95} = -q_{0.05}
$$

**Gaussian (parametric) VaR**
- Estimate mean and volatility of returns:

$$
\mu = \mathbb{E}[r], \quad \sigma = \sqrt{\text{Var}(r)}
$$

- Under
$$
\(r \sim \mathcal{N}(\mu, \sigma^2)\)
$$

the 5th percentile is:

$$
q_{0.05}^{\text{norm}} = \mu + \sigma z_{0.05}
$$

where 

$$
\(z_{0.05} = \Phi^{-1}(0.05) \approx -1.64485\).
$$

- VaR is:

$$
\text{VaR}^{\text{norm}}_{0.95} = -q_{0.05}^{\text{norm}}
$$

### 4) Rolling VaR (252 trading days) + 1-day ahead alignment
Risk is not stable over time (volatility clusters), so the script re-estimates VaR on a rolling window:
- window length: `w = 252` (≈ one trading year)

For each day \(t\), VaR is computed using information up to \(t-1\) and then **shifted by 1** so it is a genuine 1-day-ahead forecast.

Rolling historical VaR:

$$
\text{VaR}^{\text{hist}}_{t} = -\text{Quantile}_{0.05}(r_{t-w:t-1})
$$

Rolling Gaussian VaR:

$$
\text{VaR}^{\text{norm}}_{t} = -\left(\mu_{t-w:t-1} + \sigma_{t-w:t-1} z_{0.05}\right)
$$

### 5) Backtest (exceptions rate)
An exception is counted when the realized return is below the VaR cutoff:

$$
\text{exception at } t \iff r_t < -\text{VaR}_t
$$

At 95% VaR, the expected exception rate is about **5%**.  
The script prints:
- expected exception rate (5%)
- realized exception rate for historical VaR
- realized exception rate for Gaussian VaR

### 6) Plots
The script produces two plots:
1) **Rolling VaR time series** (historical vs Gaussian)
2) **Histogram of returns** with both 5% cutoff lines marked

---

## When the two VaRs differ (intuition)
- If returns have **fat left tails** (crashes more frequent than Normal), **historical VaR** is often larger (more conservative).
- If the sample includes crisis periods, the empirical 5% quantile can move a lot, while the Normal model compresses everything into \(\mu,\sigma\).
- If returns are skewed (common for equities), the Gaussian model misses asymmetry by construction.

---

## How to run

```bash
pip install yfinance matplotlib numpy
python var_engine.py
