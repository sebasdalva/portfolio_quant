# Portfolio Quant — Guía para Claude Code

## Estado actual (2026-06-24)

Proyecto migrado completamente de Google Colab/Drive a entorno local con `.venv` y git.
Tres notebooks operativos en secuencia. Cinco estrategias activas.

## Flujo de trabajo

```
research.ipynb → signals.ipynb → backtest.ipynb
```

Correr siempre en ese orden. Cada notebook es independiente salvo esa dependencia de archivos.

## Cómo ejecutar

```bash
# Activar entorno
.venv\Scripts\activate          # Windows
source .venv/bin/activate       # Mac/Linux

# Ejecutar un notebook
jupyter nbconvert --to notebook --execute --inplace notebooks/research.ipynb
jupyter nbconvert --to notebook --execute --inplace notebooks/signals.ipynb
jupyter nbconvert --to notebook --execute --inplace notebooks/backtest.ipynb
```

## Estructura de archivos clave

```
portfolio_quant/
├── CLAUDE.md                          ← este archivo
├── portfolio_quant_project.md         ← doc maestro del proyecto
├── pyproject.toml                     ← dependencias (uv)
├── config/
│   ├── universe.csv                   ← tickers activos (active=yes)
│   ├── costs_rates.csv                ← rf=2%, costos por defecto
│   └── rules.yaml                     ← ma_length=15, apply_fraction=1/6
├── data/
│   ├── prices/<TICKER>.csv            ← OHLCV crudo, append-only
│   ├── prices_adjusted.csv            ← panel ajustado (output de research)
│   └── signals/
│       ├── daily_returns_STRAT001.csv ← output de signals
│       ├── daily_returns_STRAT002.csv
│       ├── daily_returns_STRAT003.csv
│       ├── daily_returns_STRAT004.csv
│       ├── daily_returns_STRAT005.csv
│       └── sharpe_ratios_STRAT005.csv
├── reports/
│   └── backtest_report_YYYY-MM-DD.html  ← reporte HTML (output de backtest)
└── notebooks/
    ├── research.ipynb    ← descarga precios, construye prices_adjusted.csv
    ├── signals.ipynb     ← genera daily_returns_STRATXXX.csv por estrategia
    ├── backtest.ipynb    ← métricas IS/OOS, visualización, reporte HTML
    └── old_*/            ← notebooks archivados (no usar)
```

## Estrategias activas

| ID | Descripción | Señal |
|---|---|---|
| STRAT001 | Return Stacking + Hedge Gradual (RSSB/RSST/GLD/SLV/TIP/HYG/PFF) | MA gradual 1/6 |
| STRAT002 | 60/40 SPY/AGG (benchmark) | Siempre largo |
| STRAT003 | Trend Following binario SPY | MA binaria |
| STRAT004 | Permanent Portfolio 25/25/25/25 (benchmark) | Siempre largo |
| STRAT005 | Dual Momentum cross-sectional (Sharpe ranking, top-5) | Siempre largo |

Para agregar STRAT006+: agregar celda en `signals.ipynb` y entrada en `STRAT_LABELS` de `backtest.ipynb`. El resto es automático.

## Patrón de paths (todos los notebooks)

```python
from pathlib import Path

def find_repo_root(marker='pyproject.toml'):
    current = Path.cwd().resolve()
    for candidate in [current, *current.parents]:
        if (candidate / marker).exists():
            return candidate
    raise FileNotFoundError(f"No se encontró '{marker}'")

BASE = find_repo_root()
```

Sin rutas hardcodeadas. Sin `google.colab`. Funciona en Windows y Mac.

## Convenciones

- **No look-ahead**: señales en t-1, posición en t. Shift explícito en todas las funciones de señal.
- **Split IS/OOS**: 80/20 por defecto. No modificar OOS una vez definido.
- **CSVs de señales**: columnas `date` (index) + `daily_return`. Un archivo por estrategia.
- **n_trials_total en DSR**: incrementar cada vez que se re-testea una variante sobre los mismos datos IS.
