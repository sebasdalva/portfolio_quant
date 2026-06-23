# MScFE Programming Style & Quantitative Finance SKILL

**title:** MScFE Programming Standards & Code Reuse Guide  
**purpose:** Authoritative reference for coding style, design patterns, and computational discipline learned in WQU MScFE (652 Portfolio Mgmt, 632 ML, 620 Derivatives, 610 Econometrics).  
**scope:** Python/R, financial data workflows, backtesting, optimization, time-series, matrix operations.  
**how_to_use:** When starting any new project—portfolio, ML model, backtest, risk analysis—consult the relevant section first. Code blocks are production-ready (copy-paste). Follow naming conventions and structure religiously. When in doubt, refer to **EXAM INSIGHT** traps to avoid common errors.

---

## PHILOSOPHY: "RUN CODE FIRST, THEN STATE THE VERDICT"

**Core principle from MScFE 632 (ML) & MScFE 652 (Portfolio):**
- Never answer numerical questions from memory or "by inspection."
- For any calculation (Sharpe ratio, Gini, kelly weights, factor counts, etc.), **write code, run it, read the output, then explain.**
- Quiz traps exploit edge cases, loop order, and definition boundary conditions that only execution reveals.
- Hand arithmetic is error-prone; reliance on it in production portfolios is career-ending.

---

## SECTION 1: NAMING CONVENTIONS & STRUCTURE

### 1.1 Variable Names (Python)

```python
# GOOD: descriptive, follows pep8, domain-clear
returns_daily          # 1D array of periodic returns
cov_matrix             # covariance (numpy 2D)
w_kelly                # portfolio weights (Kelly allocation)
ma_90                  # moving average (90-day)
rf_annual              # risk-free rate (annual, float)
sharpe_ratio_annual    # Sharpe annualized (float)
n_assets, n_periods    # counts (int)
eVal, eVec             # eigenvalues, eigenvectors (from PCA/spectral)
hedge_fraction         # fractional size for hedging (0.0–1.0)

# AVOID: cryptic, ambiguous, not domain-specific
ret, r, rts            # unclear: daily? monthly? simple? log?
sig                    # sigma? Is it volatility or std?
w                      # weight of what? Full portfolio? Single asset?
x, y, z                # no financial meaning
data                   # too vague (prices? returns? both?)
```

### 1.2 Project Structure

```
portfolio_quant_project/
├── data/
│   ├── fetch_prices.py          # yfinance, pandas_datareader downloads
│   ├── clean_returns.py         # log vs simple, remove NaNs, split train/test
│   └── market_data.csv          # cache of historical prices
│
├── signals/
│   ├── momentum.py              # MA90, trend filters, hedge signals
│   ├── sizing.py                # kelly_weights(), 1/6 allocations, caps
│   └── execution.py             # order placement (if live)
│
├── backtest/
│   ├── portfolio_backtest.py    # vectorized returns, cumulative PnL
│   ├── attribution.py           # by-source breakdown (equity, bond, trend, etc.)
│   └── metrics.py               # Sharpe, drawdown, VaR, Sortino
│
├── live/
│   ├── weekly_signal.py         # compute next week's hedge adjustment
│   └── trade_log.md             # manual log (YYYY-MM-DD, signals, execution)
│
├── tests/
│   ├── test_momentum.py         # unit tests for signal logic
│   └── test_kelly.py            # validate optimization weights
│
├── notebooks/
│   └── analysis.ipynb           # exploratory work, prototyping
│
├── config.py                    # constants, thresholds, instrument universes
├── requirements.txt             # numpy, pandas, scipy, yfinance, etc.
└── README.md                    # project overview, how to run

# KEY PRINCIPLE: Separate data fetch → signal compute → backtest → visualization
# Each layer should be testable independently.
```

### 1.3 Function Signatures (Python)

```python
# GOOD: clear docstring, type hints (optional but encouraged), defaults sensible

def annualized_sharpe(returns, rf_annual=0.05, periods=252):
    """
    Compute annualized Sharpe ratio.
    
    Args:
        returns (np.ndarray): 1D array of periodic (e.g., daily) simple returns.
        rf_annual (float): Annual risk-free rate. Default 5% (0.05).
        periods (int): Trading days per year. Default 252.
    
    Returns:
        float: Annualized Sharpe ratio.
    
    Note:
        Assumes returns are simple (1 + daily_pct_change), not log returns.
        Does NOT assume drift in rf; uses rf/periods for daily subtraction.
    """
    r = np.asarray(returns, dtype=float)
    excess = r - rf_annual / periods
    return np.sqrt(periods) * excess.mean() / r.std(ddof=1)


def kelly_weights(mu_excess, cov, fraction=1.0, weight_cap=0.20):
    """
    Compute Kelly-optimal portfolio weights with fractional scaling and per-asset caps.
    
    Args:
        mu_excess (np.ndarray): 1D array of expected excess returns.
        cov (np.ndarray): 2D covariance matrix.
        fraction (float): Scaling factor (0.5 = half-Kelly, 2.0 = double). Default 1.0.
        weight_cap (float): Maximum absolute weight per asset. Default 0.20 (20%).
    
    Returns:
        np.ndarray: 1D weight vector (may not sum to 1 if capped).
    
    Raises:
        np.linalg.LinAlgError: If covariance matrix is singular.
    
    EXAM INSIGHT (M5L1): Full Kelly maximizes log-growth; fractional Kelly trades
    growth for lower drawdown. Double-Kelly (fraction=2.0) increases leverage but
    volatility.
    """
    w = fraction * np.linalg.inv(cov) @ mu_excess
    w = np.clip(w, -weight_cap, weight_cap)
    return w

# AVOID: vague docstrings, missing type info, no error handling
def compute_stuff(x, y):
    return x @ np.linalg.inv(y)  # what is x? y? dimensions? singular risk?
```

---

## SECTION 2: DATA WORKFLOWS & PREPROCESSING

### 2.1 Fetching & Cleaning Returns

```python
import numpy as np
import pandas as pd
import yfinance as yf
from datetime import datetime

def fetch_prices(tickers, start_date, end_date, dropna=True):
    """
    Download adjusted close prices from Yahoo Finance.
    
    Args:
        tickers (list): List of ticker symbols.
        start_date, end_date (str): ISO format 'YYYY-MM-DD'.
        dropna (bool): Drop rows with any NaN. Default True.
    
    Returns:
        pd.DataFrame: Adjusted close prices (dates as index, tickers as columns).
    """
    data = yf.download(tickers, start=start_date, end=end_date, progress=False)
    if isinstance(data, pd.Series):  # single ticker edge case
        data = data.to_frame(tickers[0])
    if dropna:
        data = data.dropna()
    return data


def compute_returns(prices, method='simple'):
    """
    Compute periodic returns from price series.
    
    Args:
        prices (pd.DataFrame or pd.Series): Price data (dates as index).
        method (str): 'simple' or 'log'. Default 'simple'.
    
    Returns:
        pd.DataFrame or pd.Series: Returns (first row is NaN, removed after).
    
    Note:
        Simple: R_t = (P_t / P_{t-1}) - 1
        Log:    R_t = ln(P_t / P_{t-1})
        For backtests, simple returns are standard in finance.
    """
    if method == 'simple':
        returns = prices.pct_change()
    elif method == 'log':
        returns = np.log(prices / prices.shift(1))
    else:
        raise ValueError(f"method must be 'simple' or 'log', got {method}")
    return returns.dropna()


def train_test_split_ts(returns, train_ratio=0.8, shuffle=False):
    """
    Time-series aware train-test split (NO shuffling for time-series data).
    
    Args:
        returns (pd.DataFrame): Return data (dates as index).
        train_ratio (float): Fraction of data for training. Default 0.8.
        shuffle (bool): MUST be False for time-series. Raises error if True.
    
    Returns:
        tuple: (returns_train, returns_test)
    
    EXAM INSIGHT (M8L2): Shuffling time-series breaks temporal structure and
    causes look-ahead bias. Always use shuffle=False; if you need cross-validation,
    use TimeSeriesSplit or purged k-fold from sklearn.
    """
    if shuffle:
        raise ValueError("shuffle=True breaks time-series structure. Use shuffle=False.")
    n = len(returns)
    split_idx = int(n * train_ratio)
    return returns.iloc[:split_idx], returns.iloc[split_idx:]


# Example usage
prices = fetch_prices(['RSSB', 'RSST', 'RSIT', 'GLD', 'SLV'],
                      start_date='2023-01-01', end_date='2025-09-30')
returns = compute_returns(prices, method='simple')
ret_train, ret_test = train_test_split_ts(returns, train_ratio=0.8)

print(f"Training period: {ret_train.index[0]} to {ret_train.index[-1]} ({len(ret_train)} days)")
print(f"Test period:     {ret_test.index[0]} to {ret_test.index[-1]} ({len(ret_test)} days)")
```

### 2.2 Annualization & Compounding

```python
TRADING_DAYS = 252  # NYSE annual trading days (standard)

def annualize_metric(daily_metric, metric_name='return'):
    """
    Annualize a daily metric.
    
    Args:
        daily_metric (float or np.ndarray): Daily-frequency statistic.
        metric_name (str): 'return', 'volatility', or 'sharpe'. Scaling differs.
    
    Returns:
        float or np.ndarray: Annualized metric.
    """
    if metric_name == 'return':
        return (1 + daily_metric) ** TRADING_DAYS - 1
    elif metric_name == 'volatility':
        return daily_metric * np.sqrt(TRADING_DAYS)
    elif metric_name == 'sharpe':
        return daily_metric * np.sqrt(TRADING_DAYS)
    else:
        raise ValueError(f"metric_name {metric_name} not recognized")


def compound_returns(daily_returns):
    """
    Compute cumulative return from daily returns (geometric compounding).
    
    Args:
        daily_returns (np.ndarray or pd.Series): Daily simple returns.
    
    Returns:
        float: Total return (cumulative).
    """
    return np.prod(1 + daily_returns) - 1
```

---

## SECTION 3: PORTFOLIO OPTIMIZATION (MScFE 652 Pattern)

### 3.1 Mean-Variance Optimization (MVP)

```python
from scipy.optimize import minimize

def optimize_mvp(mu, cov, target_return=None, weight_bounds=(0, 1), 
                 constraints=None, max_weight=None):
    """
    Solve mean-variance portfolio problem.
    
    min w^T Σ w  subject to:
    - w^T 1 = 1
    - w_i ∈ [L_i, U_i]  (weight bounds, typically [0, max_weight])
    - (optional) w^T μ ≥ target_return
    
    Args:
        mu (np.ndarray): 1D array of expected returns.
        cov (np.ndarray): 2D covariance matrix.
        target_return (float): Min portfolio return constraint. Optional.
        weight_bounds (tuple): (lower, upper) per-asset bounds. Default (0, 1).
        max_weight (float): Override both bounds to be (0, max_weight).
        constraints (list): Additional scipy constraint dicts.
    
    Returns:
        np.ndarray: Optimal weights.
    
    EXAM INSIGHT (M3L1–M3L2): Sum-to-one is enforced by constraint.
    If Σ upper_bounds < 1.0, the problem is infeasible → no solution exists.
    """
    n = len(mu)
    
    if max_weight is not None:
        bounds = [(0, max_weight)] * n
    else:
        bounds = [weight_bounds] * n
    
    # Default constraints: sum to 1
    if constraints is None:
        constraints = []
    constraints.append({'type': 'eq', 'fun': lambda w: w.sum() - 1})
    
    # Optional: target return
    if target_return is not None:
        constraints.append({'type': 'ineq', 'fun': lambda w: mu @ w - target_return})
    
    # Initial guess: equal weight
    w0 = np.ones(n) / n
    
    # Minimize portfolio variance
    result = minimize(
        lambda w: w @ cov @ w,  # objective: variance
        w0,
        method='SLSQP',
        bounds=bounds,
        constraints=constraints
    )
    
    if not result.success:
        print(f"Optimization failed: {result.message}")
        return None
    
    return result.x


def efficient_frontier(mu, cov, n_points=50, bounds=(0, 1)):
    """
    Trace the efficient frontier (risk-return curve).
    
    Returns:
        tuple: (portfolio_returns, portfolio_risks, weights_list)
    """
    min_ret, max_ret = mu.min(), mu.max()
    target_returns = np.linspace(min_ret, max_ret, n_points)
    
    frontier_ret, frontier_risk, frontier_weights = [], [], []
    
    for target in target_returns:
        w = optimize_mvp(mu, cov, target_return=target, weight_bounds=bounds)
        if w is not None:
            port_ret = mu @ w
            port_risk = np.sqrt(w @ cov @ w)
            frontier_ret.append(port_ret)
            frontier_risk.append(port_risk)
            frontier_weights.append(w)
    
    return np.array(frontier_ret), np.array(frontier_risk), frontier_weights
```

### 3.2 Kelly Criterion

```python
def kelly_weights(mu_excess, cov, fraction=1.0, cap=None):
    """
    Kelly-optimal weights: w* = (1/λ) Σ^{-1} μ_excess
    where λ is the risk-aversion parameter (implicit in Σ).
    
    For portfolio context, λ = 1 (unitless).
    
    Args:
        mu_excess (np.ndarray): Expected excess returns (above risk-free).
        cov (np.ndarray): Covariance matrix.
        fraction (float): Scaling (0.5=half-Kelly, 2.0=double). Default 1.0.
        cap (float): Clip weights to [-cap, cap]. Optional.
    
    Returns:
        np.ndarray: Kelly-optimal weights (may not sum to 1 if capped).
    """
    try:
        inv_cov = np.linalg.inv(cov)
    except np.linalg.LinAlgError:
        raise ValueError("Covariance matrix is singular; cannot invert.")
    
    w_kelly = fraction * (inv_cov @ mu_excess)
    
    if cap is not None:
        w_kelly = np.clip(w_kelly, -cap, cap)
    
    return w_kelly


def kelly_vs_mv(returns_train, returns_test, rf=0.05):
    """
    Compare Kelly-optimal, half-Kelly, and double-Kelly portfolios on test set.
    
    Returns:
        pd.DataFrame: Sharpe, return, volatility, max drawdown for each.
    """
    mu = returns_train.mean() * TRADING_DAYS
    cov = returns_train.cov() * TRADING_DAYS
    mu_excess = mu - rf
    
    # Three Kelly variants
    w_kelly_full = kelly_weights(mu_excess, cov, fraction=1.0, cap=0.20)
    w_kelly_half = kelly_weights(mu_excess, cov, fraction=0.5, cap=0.20)
    w_kelly_double = kelly_weights(mu_excess, cov, fraction=2.0, cap=0.20)
    
    results = {}
    for name, w in [('Kelly Full', w_kelly_full), 
                     ('Kelly Half', w_kelly_half), 
                     ('Kelly Double', w_kelly_double)]:
        port_ret = (returns_test @ w).values
        sharpe = annualized_sharpe(port_ret, rf_annual=rf)
        cum_ret = compound_returns(port_ret)
        vol = annualize_metric(port_ret.std(), 'volatility')
        max_dd = compute_max_drawdown(port_ret)
        
        results[name] = {
            'Sharpe': sharpe,
            'Cumulative Return': cum_ret,
            'Annualized Vol': vol,
            'Max Drawdown': max_dd
        }
    
    return pd.DataFrame(results).T
```

---

## SECTION 4: COVARIANCE IMPROVEMENT (MScFE 652 M7)

### 4.1 Marchenko-Pastur Denoising (López de Prado)

```python
from sklearn.neighbors import KernelDensity
from scipy.optimize import minimize

def mpPDF(var, q, pts=1000):
    """
    Marchenko-Pastur probability density function.
    q = T / N (ratio of observations to variables).
    """
    eMin = var * (1 - (1.0 / q) ** 0.5) ** 2
    eMax = var * (1 + (1.0 / q) ** 0.5) ** 2
    eVal = np.linspace(eMin, eMax, pts)
    pdf = q / (2 * np.pi * var * eVal) * np.sqrt((eMax - eVal) * (eVal - eMin))
    return pd.Series(pdf, index=eVal)


def fitKDE(obs, bWidth=0.15):
    """Fit kernel density estimator to eigenvalue distribution."""
    if len(obs.shape) == 1:
        obs = obs.reshape(-1, 1)
    kde = KernelDensity(kernel='gaussian', bandwidth=bWidth).fit(obs)
    x = np.unique(obs).reshape(-1, 1)
    logProb = kde.score_samples(x)
    return pd.Series(np.exp(logProb), index=x.flatten())


def findMaxEval(eVal, q, bWidth=0.15):
    """Find the noise eigenvalue threshold using MP pdf."""
    def errPDFs(var):
        pdf0 = mpPDF(var[0], q, pts=1000)
        pdf1 = fitKDE(eVal, bWidth=bWidth)
        pdf1 = pdf1.reindex(pdf0.index, fill_value=0)
        return np.sum((pdf1 - pdf0) ** 2)
    
    out = minimize(errPDFs, x0=np.array(1.0), bounds=((1e-5, 1-1e-5),))
    var = out.x[0] if out.success else 1.0
    eMax = var * (1 + (1.0 / q) ** 0.5) ** 2
    return eMax, var


def denoisedCov(cov, n_facts=1):
    """
    Replace noise eigenvalues with their average (replaces diagonal of noise portion).
    
    Args:
        cov (np.ndarray): Sample covariance matrix.
        n_facts (int): Number of principal factors to keep. Default 1.
    
    Returns:
        np.ndarray: Denoised covariance (same shape, reduced rank).
    """
    eVal, eVec = np.linalg.eigh(cov)
    idx = eVal.argsort()[::-1]
    eVal, eVec = eVal[idx], eVec[:, idx]
    
    q = cov.shape[0] / cov.shape[1]  # T / N
    eMax, _ = findMaxEval(eVal, q)
    
    # Replace noise eigenvalues with their mean
    eVal_copy = eVal.copy()
    eVal_copy[n_facts:] = eVal_copy[n_facts:].mean()
    
    # Reconstruct
    cov_denoised = eVec @ np.diag(eVal_copy) @ eVec.T
    return cov_denoised


# EXAM INSIGHT (M7L2): Eigenvalues below λ+ = σ²(1+√(N/T))² are noise.
# Averaging them reduces estimation error but loses rank.
```

### 4.2 Hierarchical Risk Parity (López de Prado)

```python
from scipy.cluster.hierarchy import linkage, dendrogram

def getIVP(cov):
    """Inverse-variance portfolio (equal risk contribution)."""
    ivp = 1.0 / np.diag(cov)
    return ivp / ivp.sum()


def getClusterVar(cov, cluster_items, asset_names=None):
    """Variance of a cluster (sub-portfolio)."""
    cov_cluster = cov.loc[cluster_items, cluster_items]
    w = getIVP(cov_cluster.values).reshape(-1, 1)
    return (w.T @ cov_cluster.values @ w)[0, 0]


def getQuasiDiag(linkage_matrix):
    """Reorder assets to place high covariances on diagonal."""
    link = linkage_matrix.astype(int)
    sortIx = pd.Series([link[-1, 0], link[-1, 1]])
    numItems = link[-1, 3]
    
    while sortIx.max() >= numItems:
        sortIx.index = range(0, sortIx.shape[0] * 2, 2)
        df0 = sortIx[sortIx >= numItems]
        i = df0.index
        j = df0.values - numItems
        sortIx[i] = link[j, 0]
        df0 = pd.Series(link[j, 1], index=i + 1)
        sortIx = pd.concat([sortIx, df0]).sort_index()
        sortIx.index = range(sortIx.shape[0])
    
    return sortIx.tolist()


def HRP(cov, asset_names):
    """
    Hierarchical Risk Parity portfolio.
    
    Args:
        cov (pd.DataFrame): Covariance matrix (indexed by asset names).
        asset_names (list): Asset identifiers.
    
    Returns:
        pd.Series: HRP weights.
    """
    # Step 1: Linkage (hierarchical clustering)
    corr = cov.corr()
    dist = ((1 - corr) / 2) ** 0.5
    link = linkage(dist.values[np.triu_indices_from(dist.values, k=1)], method='ward')
    
    # Step 2: Quasi-diagonalization
    sortIx = getQuasiDiag(link)
    
    # Step 3: Recursive bisection
    w = pd.Series(1.0, index=sortIx)
    cItems = [sortIx]
    
    while len(cItems) > 0:
        cItems = [i[j:k] for i in cItems
                  for j, k in ((0, len(i)//2), (len(i)//2, len(i))) if len(i) > 1]
        
        for i in range(0, len(cItems), 2):
            c0, c1 = cItems[i], cItems[i+1]
            v0 = getClusterVar(cov, c0)
            v1 = getClusterVar(cov, c1)
            alpha = 1 - v0 / (v0 + v1)
            w[c0] *= alpha
            w[c1] *= 1 - alpha
    
    return w


# EXAM INSIGHT (M7L4): HRP avoids matrix inversion → robust to ill-conditioning.
# Allocates MORE weight to lower-risk assets (risk parity principle).
```

---

## SECTION 5: BACKTESTING & PERFORMANCE METRICS (MScFE 652 & 632)

### 5.1 Portfolio Backtest (Vectorized)

```python
def portfolio_backtest(returns, weights_over_time=None, fixed_weights=None, 
                       rebalance_freq='D'):
    """
    Vectorized portfolio backtest (assumes weights constant or rebalanced).
    
    Args:
        returns (pd.DataFrame): Daily returns (dates as index, assets as columns).
        weights_over_time (pd.DataFrame): Time-varying weights (dates as index).
        fixed_weights (np.ndarray or dict): Fixed 1D weight vector.
        rebalance_freq (str): 'D' (daily), 'W' (weekly), 'M' (monthly). Default 'D'.
    
    Returns:
        pd.Series: Cumulative portfolio returns (indexed by date).
    """
    if fixed_weights is not None:
        if isinstance(fixed_weights, dict):
            w = pd.Series(fixed_weights)
        else:
            w = pd.Series(fixed_weights, index=returns.columns)
        port_returns = (returns @ w).squeeze()
    elif weights_over_time is not None:
        # Apply weights that vary by date
        port_returns = (returns * weights_over_time).sum(axis=1)
    else:
        raise ValueError("Either fixed_weights or weights_over_time must be provided.")
    
    cumulative = (1 + port_returns).cumprod() - 1
    return cumulative


def compute_max_drawdown(returns):
    """
    Maximum drawdown from cumulative returns.
    
    Args:
        returns (np.ndarray): Periodic returns (simple, NOT cumulative).
    
    Returns:
        float: Maximum drawdown (negative, e.g., -0.25 for 25% DD).
    """
    cumulative = np.cumprod(1 + returns)
    running_max = np.maximum.accumulate(cumulative)
    dd = (cumulative - running_max) / running_max
    return dd.min()


def performance_summary(returns, rf=0.05, name='Portfolio'):
    """
    Compute key performance metrics.
    
    Returns:
        pd.Series: Sharpe, total return, volatility, max DD, Sortino, etc.
    """
    total_ret = compound_returns(returns)
    ann_ret = annualize_metric(returns.mean(), 'return')
    ann_vol = annualize_metric(returns.std(), 'volatility')
    sharpe = annualized_sharpe(returns, rf_annual=rf)
    max_dd = compute_max_drawdown(returns)
    
    # Sortino: downside-only volatility
    downside = returns[returns < 0]
    downside_vol = annualize_metric(downside.std(), 'volatility') if len(downside) > 0 else 0
    sortino = (ann_ret - rf) / downside_vol if downside_vol > 0 else 0
    
    return pd.Series({
        'Total Return': total_ret,
        'Annualized Return': ann_ret,
        'Annualized Vol': ann_vol,
        'Sharpe Ratio': sharpe,
        'Sortino Ratio': sortino,
        'Max Drawdown': max_dd,
        'Calmar Ratio': ann_ret / abs(max_dd) if max_dd != 0 else np.inf
    }, name=name)
```

### 5.2 Deflated Sharpe Ratio (Selection Bias Correction)

```python
from scipy.stats import norm

def deflated_sharpe(sr_observed, n_trials, sr_variance, T, skew=0, kurt=3):
    """
    Deflated Sharpe Ratio: adjusts observed Sharpe for multiple comparisons.
    
    Args:
        sr_observed (float): Best observed Sharpe across trials (per-period).
        n_trials (int): Number of trial strategies tested.
        sr_variance (float): Variance of Sharpe estimates across trials.
        T (int): Number of observations in backtest.
        skew (float): Skewness of returns. Default 0 (normal).
        kurt (float): Excess kurtosis. Default 3 (leptokurtic).
    
    Returns:
        float: Probability that true SR > 0 after deflation.
    
    EXAM INSIGHT (M8L1): Picking the best of N strategies inflates Sharpe.
    Deflation corrects for this selection bias using extreme value theory.
    """
    emc = 0.5772156649  # Euler-Mascheroni constant
    
    # Expected maximum under null (no true edge)
    z_left = norm.ppf(1 - 1.0 / n_trials)
    z_right = norm.ppf(1 - 1.0 / (n_trials * np.e))
    sr_max_expected = sr_variance ** 0.5 * ((1 - emc) * z_left + emc * z_right)
    
    # Deflate
    numerator = (sr_observed - sr_max_expected) * np.sqrt(T - 1)
    denominator = np.sqrt(1 - skew * sr_observed + (kurt - 1) / 4.0 * sr_observed ** 2)
    
    return norm.cdf(numerator / denominator)
```

---

## SECTION 6: TIME-SERIES & MOMENTUM FEATURES (MScFE 632 M1)

### 6.1 Feature Engineering (Momentum)

```python
def rolling_compound_return(prices, window=25, in_percent=True):
    """
    Geometric mean return over rolling window.
    
    Args:
        prices (pd.Series): Price series.
        window (int): Lookback window (days).
        in_percent (bool): Return as %. Default True.
    
    Returns:
        pd.Series: Compounded return over window.
    
    Formula:
        R_compound = (P_t / P_{t-window})^(1/window) - 1
    """
    returns = prices.pct_change()
    comp = returns.rolling(window).apply(
        lambda x: (np.prod(1 + x) ** (1 / window) - 1) * (100 if in_percent else 1)
    )
    return comp


def create_momentum_features(prices, windows=[10, 25, 60, 120, 240]):
    """
    Create lagged compounded-return features for momentum trading.
    
    Returns:
        pd.DataFrame: Features (one column per window).
    """
    features = pd.DataFrame(index=prices.index)
    for w in windows:
        features[f'Mom_{w}d'] = rolling_compound_return(prices, window=w)
    return features


def create_target(returns, horizon=25):
    """
    Create target: realized return over next 'horizon' days.
    
    Args:
        returns (pd.Series): Periodic returns.
        horizon (int): Forecast horizon (days ahead). Default 25.
    
    Returns:
        pd.Series: Forward-looking return (shifted).
    """
    return returns.rolling(horizon).apply(
        lambda x: (np.prod(1 + x) - 1) * 100  # as percentage
    ).shift(-horizon)  # shift so target aligns with X_t


# EXAMPLE: Prepare data for ML
prices = yf.download('SPY', start='2000-01-01', end='2022-01-01')['Adj Close']
returns = prices.pct_change().dropna()

X = create_momentum_features(prices)
y = create_target(returns, horizon=25)

# Align and drop NaNs
df = pd.concat([X, y], axis=1).dropna()
X_clean = df.iloc[:, :-1]
y_clean = df.iloc[:, -1]

X_train = X_clean.iloc[:int(0.8*len(X_clean))]
y_train = y_clean.iloc[:int(0.8*len(y_clean))]
X_test = X_clean.iloc[int(0.8*len(X_clean)):]
y_test = y_clean.iloc[int(0.8*len(y_clean)):]
```

---

## SECTION 7: MACHINE LEARNING DISCIPLINE (MScFE 632)

### 7.1 Preprocessing & Scaling

```python
from sklearn.preprocessing import StandardScaler, MinMaxScaler
from sklearn.model_selection import TimeSeriesSplit

def preprocess_features(X_train, X_test, method='standard', feature_range=(-1, 1)):
    """
    Scale features on training data; apply same transform to test.
    
    Args:
        X_train, X_test (pd.DataFrame): Feature matrices.
        method (str): 'standard' (mean=0, std=1) or 'minmax' (range).
        feature_range (tuple): For minmax, target range. Default (-1, 1).
    
    Returns:
        tuple: (X_train_scaled, X_test_scaled, scaler)
    
    CRITICAL: Fit scaler ONLY on training data, then transform both train+test.
    Do NOT fit on test data (causes leakage).
    """
    if method == 'standard':
        scaler = StandardScaler()
    elif method == 'minmax':
        scaler = MinMaxScaler(feature_range=feature_range)
    else:
        raise ValueError(f"method {method} not recognized")
    
    X_train_scaled = scaler.fit_transform(X_train)
    X_test_scaled = scaler.transform(X_test)
    
    return pd.DataFrame(X_train_scaled, index=X_train.index, columns=X_train.columns), \
           pd.DataFrame(X_test_scaled, index=X_test.index, columns=X_test.columns), \
           scaler


def time_series_cv_split(X, y, n_splits=5):
    """
    Walk-forward time-series cross-validation (non-anchored splits).
    
    Returns:
        generator of (train_idx, test_idx) tuples
    
    NOTE: Uses sklearn.model_selection.TimeSeriesSplit (no shuffle).
    """
    tscv = TimeSeriesSplit(n_splits=n_splits)
    for train_idx, test_idx in tscv.split(X):
        yield train_idx, test_idx
```

### 7.2 Classification Metrics

```python
def confusion_matrix_metrics(y_true, y_pred):
    """
    Compute TP, TN, FP, FN and derived metrics (Precision, Recall, F1, Gini).
    
    Args:
        y_true (np.ndarray): Ground truth (0 or 1).
        y_pred (np.ndarray): Predicted class (0 or 1).
    
    Returns:
        dict: Confusion matrix entries + metrics.
    """
    tp = np.sum((y_true == 1) & (y_pred == 1))
    tn = np.sum((y_true == 0) & (y_pred == 0))
    fp = np.sum((y_true == 0) & (y_pred == 1))
    fn = np.sum((y_true == 1) & (y_pred == 0))
    
    accuracy = (tp + tn) / (tp + tn + fp + fn)
    precision = tp / (tp + fp) if (tp + fp) > 0 else 0
    recall = tp / (tp + fn) if (tp + fn) > 0 else 0
    f1 = 2 * precision * recall / (precision + recall) if (precision + recall) > 0 else 0
    
    # Gini = 2*AUC - 1
    # Simplified for binary: Gini = (tp + tn - 2*fn) / (tp + tn + fp + fn)
    gini = (tp + tn - fp - fn) / (tp + tn + fp + fn)
    
    return {
        'TP': tp, 'TN': tn, 'FP': fp, 'FN': fn,
        'Accuracy': accuracy,
        'Precision': precision,
        'Recall': recall,
        'F1': f1,
        'Gini': gini
    }
```

---

## SECTION 8: COMMON EXAM TRAPS & PITFALLS

### 8.1 Trap List (MScFE 652 & 632)

| Trap | Expected Answer | Why Students Fail |
|------|-----------------|-------------------|
| **M²** (Risk-adj perf) | M² = rf + SR_p · σ_market | Forget to add rf at the end |
| **Coskewness sign** | Negative = BAD for portfolio | Assume all skewness is bad |
| **IRP Forward** | F = S · (1+r_for)/(1+r_dom) | Flip numerator/denominator |
| **BL Equilibrium** | Objective (market-cap based) | Call it "manager's view" |
| **CLA Infeasibility** | No solution if Σ u_i < 1 | Say "set all to upper bounds" |
| **Factor param count** | K + N + NK + N + K(K+1)/2 | Forget the K factor means |
| **Kelly Growth** | Maximizes log(wealth) | Confuse with variance minimization |
| **HRP Inversion** | NO matrix inversion | Assume HRP inverts covariance |
| **Denoising** | Replace noise eigenvalues with mean | Keep all eigenvalues unchanged |
| **Sharpe Annualization** | √252 · daily Sharpe | Use daily Sharpe directly |
| **Time-series Shuffle** | shuffle=False ALWAYS | Break temporal structure |
| **Scaler Leakage** | Fit only on TRAIN | Fit on train+test combined |
| **Deflated Sharpe** | Account for n_trials | Use raw (inflated) Sharpe |

### 8.2 Code Traps

```python
# TRAP 1: Forgetting to annualize
sharpe_daily = 0.05  # 5% daily Sharpe (nonsensical)
sharpe_annual_WRONG = sharpe_daily  # ❌ No annualization
sharpe_annual_RIGHT = sharpe_daily * np.sqrt(252)  # ✓ Correct

# TRAP 2: Shuffling time-series
from sklearn.model_selection import train_test_split
X_train, X_test, y_train, y_test = train_test_split(
    X, y, test_size=0.2, shuffle=True  # ❌ BREAKS time-series structure
)
# Correct:
X_train, X_test = X[:int(0.8*len(X))], X[int(0.8*len(X)):]
y_train, y_test = y[:int(0.8*len(y))], y[int(0.8*len(y)):]

# TRAP 3: Fitting scaler on test data
scaler = StandardScaler()
scaler.fit(X_train + X_test)  # ❌ Leakage!
X_train_scaled = scaler.transform(X_train)
X_test_scaled = scaler.transform(X_test)

# Correct:
scaler = StandardScaler()
X_train_scaled = scaler.fit_transform(X_train)  # ✓ Fit ONLY on train
X_test_scaled = scaler.transform(X_test)        # Then transform test

# TRAP 4: Sum-to-one constraint without checking feasibility
upper_bounds = [0.15, 0.15, 0.15, 0.15]  # sum = 0.60 < 1.0
# ❌ Problem is INFEASIBLE—no portfolio can sum to 1 with these caps!
if sum(upper_bounds) < 1.0:
    print("Infeasible: sum of upper bounds < 1.0")

# TRAP 5: Using forward-looking data in backtest
target = returns.shift(-25)  # ✓ Shift target BACKWARD (future data)
# X_t aligns with y_{t+25}, no look-ahead bias

# ❌ WRONG (leakage):
target = returns.shift(25)  # This is past data!
```

---

## SECTION 9: GWP REFERENCE PATTERNS (Group Work Project Templates)

### 9.1 Portfolio Optimization GWP (MScFE 652)

**Typical structure:**
1. Download historical prices (Jan 2024–Sep 2025 for training, Oct–Dec 2025 for OOS test)
2. Compute daily simple returns
3. Estimate μ and Σ from training period
4. Optimize: MVP, Kelly (Full/Half/Double with 20% cap)
5. Evaluate on test period: Sharpe, return, vol, max DD
6. Apply improvements: denoising, clustering (HRP), detoning
7. Backtest: compare metrics across methods

**Key constants for GWP2:**
```python
TRAIN_START = '2024-01-01'
TRAIN_END = '2025-09-30'
TEST_START = '2025-10-01'
TEST_END = '2025-12-31'
RF_ANNUAL = 0.05  # 5% for Sharpe calculations
N_ASSETS = 10  # (AAPL, BAC, GOOG, GS, LLY, META, MRK, TSLA, WMT, XOM)
KELLY_CAP = 0.20  # 20% per-asset limit
```

### 9.2 ML Pipeline GWP (MScFE 632)

**Typical structure:**
1. Create momentum features (10d, 25d, 60d, 120d, 240d compound returns)
2. Create target (25d forward return)
3. Train/test split (time-series aware, no shuffle)
4. Scale features (StandardScaler or MinMaxScaler on train only)
5. Train models: LASSO, PCA, classification trees (3 methods, one per member)
6. Evaluate: Accuracy, Precision, Recall, F1, Gini on test set
7. Write handbook: advantages, disadvantages, equations, hyperparameters
8. Marketing section: why these methods work for finance

---

## SECTION 10: DEBUGGING CHECKLIST

When results don't match expected output:

- [ ] **Data**: Are dates aligned? Any NaNs? Check `df.isna().sum()`
- [ ] **Returns**: Simple or log? Annualized or daily? Check `returns.head()`
- [ ] **Covariance**: Is it based on returns or prices? Check `cov.shape` and `cov.values.max()`
- [ ] **Optimization**: Did it converge? Check `result.success`
- [ ] **Scaling**: Did you fit scaler on train only? Check `scaler.mean_`
- [ ] **Time-series split**: No shuffle? Check `X_train.index[-1] < X_test.index[0]`
- [ ] **Weights**: Do they sum to 1 (or close)? Check `w.sum()`
- [ ] **Annualization**: Did you multiply by √252? Check `sharpe * np.sqrt(252)`
- [ ] **Metrics**: Are returns simple (not log)? Are they in the right frequency? Check `returns.mean()`

---

## SECTION 11: REFERENCES & CANONICAL PAPERS

- López de Prado (2018): *Machine Learning for Asset Managers*. Ch. 2 (denoise/detone code).
- López de Prado (2016): *Advances in Financial Machine Learning*. Ch. 4 (backtesting discipline).
- Nanakorn & Palmgren / López de Prado: HRP implementation.
- Fama & French (2015): *Five Factor Model* (factor investing).
- Kelly (1956): *Information Rate of a Noisy Channel* (Kelly criterion origins).
- Markowitz (1952): *Portfolio Selection* (mean-variance foundation).
- Pretorius & van Zyl: Reinforcement learning for portfolio mgmt (FinRL).

---

## CHANGELOG

| Date | Update | Source |
|------|--------|--------|
| 2026-06-17 | Initial creation from MScFE 652 Agent Pack & MScFE 632 Agent Pack | WQU Materials |
| 2026-06-17 | Added Section 9 (GWP patterns) & Section 10 (debugging) | Project experience |

---

**This file is maintained as the single source of truth for coding standards across all portfolio_quant projects. Always refer here first before writing code.**
