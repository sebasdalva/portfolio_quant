# Portfolio Balanceado — Proyecto Cuantitativo (v4)

> Documento maestro. Leer primero al retomar. Ver también `CLAUDE.md` para guía técnica de Claude Code.
> Marcadores: [OK] decidido · [ ] pendiente · [!] a verificar

---

## 0. Estado del proyecto (2026-06-24)

| Area | Estado |
|---|---|
| Universo + tags | [OK] config/universe.csv (24 tickers, 17 con datos completos) |
| Fuente de datos | [OK] Yahoo Finance HTTP directo, sin yfinance, agnostico |
| Almacenamiento | [OK] un CSV por ticker en data/prices/, append-only |
| Costos/tasas | [OK] config/costs_rates.csv |
| Infra | [OK] repo Git privado (GitHub) + Claude Code local + .venv (uv) |
| Pipeline de datos | [OK] research.ipynb → prices_adjusted.csv |
| Señales (5 estrategias) | [OK] signals.ipynb → data/signals/daily_returns_STRATXXX.csv |
| Backtesting + reporte | [OK] backtest.ipynb → métricas IS/OOS + HTML en reports/ |
| Ticker intl-trend (RSIT) | [!] no figura en la familia Return Stacked actual |
| RSSB / RSST en panel | [!] faltantes en prices_adjusted.csv — datos no disponibles en Yahoo |

---

## 1. Filosofia

Return stacking con overlay de momentum para reducir drawdowns. Capital en bloques iguales (~$60k), cada uno una fuente de retorno no correlacionada. El overlay hedgea gradualmente equity y commodity cuando la senal se da vuelta, SIN liquidar las posiciones subyacentes (se shortea el ETF mas liquido y barato). Maximo ~5 trades cada 14 dias.

## 2. Estructura — 7 bloques (~$60k c/u)

| Bloque | Ejecucion | Exposicion stacked | Tipo |
|---|---|---|---|
| World EQ + Bonds | RSSB | ~$60k eq global + ~$60k bonds | Movil |
| US EQ + Trend | RSST | ~$60k S&P500 + ~$60k MF | Movil |
| Intl EQ + Trend | [!] RSIT (a confirmar) | ~$60k MSCI Intl + ~$60k MF | Movil |
| Commodity | GLD/GLDM + SLV/SIVR | ~$60k real assets | Movil |
| Real rates | TIP + IEF | ~$60k inflacion + duration | Fijo |
| Corp HY + EM Sov | HYG + EMB | ~$60k credito | Fijo |
| Preferred | PFF / preferreds NRA | ~$60k carry NRA | Fijo |

Exposicion total con ~$420k: ~$600k (ratio ~1.43x).

## 3. Senales e indicadores

- Senal base (precio): cruce por debajo de la MA corta (~15d), aplicado 1/6 escalonado (6 pasos). El escalonado hace que la salida efectiva se parezca a una MA ~90d pero sin latigazos.
- Filtro opcional: volatilidad (2do indicador).
- Hedge: short del ETF mas liquido/barato (RSST->SPY, RSIT->VEA/VXUS, RSSB->VT). Cash a BOXX. No se liquida el stacked.
- Cadencia: equity ~14d, commodity ~30d. Override con nota en el log.
- Parametros en config/rules.yaml.

## 4. Datos

- Un CSV por ticker: data/prices/<TICKER>.csv.
- Se guarda OHLCV crudo + dividendos + splits. El ajustado se calcula en codigo.
- Incremental: si no existe el CSV -> backfill 10y; si existe -> append del ultimo cierre.
- Agnostico a la fuente: universe.csv no asume Yahoo.

## 5. Costos y tasas (config/costs_rates.csv)

Defaults conservadores: rf 2%, short borrow 1%, comision $2/trade. Expense ratios por ETF en universe.csv.

## 6. Backtesting y metodo

- Motor returns_risk.ipynb: quantstats (tearsheets, 30+ ratios), skfolio (HRP).
- Motor gauss314 (vendor): walk-forward CV con gap IS/OOS, Markowitz, stress tests.
- Simular rebalanceos reales (14d/30d), no pesos estaticos.
- Anti-look-ahead: senal con datos hasta t-1, trade en t.
- Anti-overfitting: fijar reglas antes del OOS; pocos parametros; cambios escasos.
- Benchmark: SPY (60/40 como alternativa).

## 7. Infraestructura

Repo Git privado en GitHub (`sebasdalva/portfolio_quant`). Entorno local con `.venv` (uv). Claude Code como asistente principal.

## 8. Estructura de la carpeta

```
portfolio_quant/
├── CLAUDE.md                          ← guía técnica para Claude Code
├── portfolio_quant_project.md         ← este doc
├── pyproject.toml                     ← dependencias (uv)
├── config/
│   ├── universe.csv
│   ├── costs_rates.csv
│   └── rules.yaml
├── data/
│   ├── prices/<TICKER>.csv            ← OHLCV crudo por ticker (append-only)
│   ├── prices_adjusted.csv            ← panel ajustado (output de research)
│   └── signals/
│       ├── daily_returns_STRAT001.csv
│       ├── ...
│       └── sharpe_ratios_STRAT005.csv
├── reports/
│   └── backtest_report_YYYY-MM-DD.html
└── notebooks/
    ├── research.ipynb    ← baja/actualiza precios, construye prices_adjusted.csv
    ├── signals.ipynb     ← construye daily_returns_STRATXXX.csv por estrategia
    ├── backtest.ipynb    ← métricas IS/OOS, gráficos, reporte HTML
    └── old_*/            ← versiones archivadas (no usar)
```

## 9. Flujo de trabajo (notebooks)

### research.ipynb — correr primero
1. `find_repo_root()` para resolver paths.
2. Leer `config/universe.csv` para saber qué tickers están activos.
3. Descargar o actualizar cada ticker (append-only, `data/prices/<TICKER>.csv`).
4. Construir `data/prices_adjusted.csv` (ajuste por dividendos y splits, método CRSP).
5. Verificar integridad del panel (NaNs, cobertura por ticker).

### signals.ipynb — depende de research
Auto-detecta `prices_adjusted.csv`. Genera una celda por estrategia:
- STRAT001: Return Stacking + Hedge Gradual (MA gradual 1/6)
- STRAT002: 60/40 SPY/AGG (benchmark, siempre largo)
- STRAT003: Trend Following binario SPY (MA binaria)
- STRAT004: Permanent Portfolio 25/25/25/25 (benchmark)
- STRAT005: Dual Momentum cross-sectional (Sharpe ranking, top-5, filtro vs SHY)

Para agregar STRAT006+: agregar celda en signals.ipynb y entrada en STRAT_LABELS de backtest.ipynb.

### backtest.ipynb — depende de signals
Auto-descubre todos los `data/signals/daily_returns_*.csv`. Produce:
- Tabla IS/OOS: CAGR, Vol, Sharpe, Sortino, Calmar, MaxDD, Win%
- Risk measures: VaR/CVaR 95/99, skewness, kurtosis
- Retornos anuales por estrategia
- Correlación entre estrategias (tabla + heatmap)
- Equity curves + drawdown + rolling Sharpe (63d)
- Deflated Sharpe Ratio (actualizar n_trials_total al re-testear)
- Reporte HTML auto-contenido en `reports/`

## 10. Parametros clave del backtest (NO tocar sin deliberacion)

| Parametro | Valor | Descripcion |
|---|---|---|
| LOOKBACK | 90 | dias para la MA de momentum |
| STEP | 1/6 | fraccion de hedge por rebalanceo |
| FREQ_EQ | 14 | dias entre rebalanceos equity |
| FREQ_COM | 30 | dias entre rebalanceos commodity |
| BENCHMARK | SPY | referencia para quantstats |

## 11. Fixed income — reglas NRA

Preferred/corporativos deben calificar para cero retencion federal (portfolio interest NRA). REIT/BDC preferreds: 30% retencion -> evitar. Exchange-traded debt de C-corps: verificar prospecto en EDGAR.

## 12. Como retomar

```
git clone https://github.com/sebasdalva/portfolio_quant.git
cd portfolio_quant
uv sync                    # instala dependencias desde pyproject.toml
```

Leer: `CLAUDE.md` → este `.md` → `config/universe.csv` → `config/rules.yaml` → notebooks en orden.

## 13. Log de trades — formato

```
## YYYY-MM-DD — Rebalanceo [N]
Senales: RSSB v/x · RSST v/x · GLD v/x · SLV v/x
Trades: | Accion | Instrumento | Qty | Precio | Motivo |
Hedge fractions: | Asset | Hedge % |
Nota: (override, evento macro)
```

## 14. Proximos pasos sugeridos

- [ ] Correr backtest.ipynb completo y revisar el reporte HTML generado en reports/.
- [ ] Resolver por qué RSSB/RSST no están disponibles en Yahoo Finance (ticker alternativo o fuente).
- [ ] Agregar STRAT006: walk-forward CV sobre STRAT001 (gauss314 skill).
- [ ] Resolver ticker RSIT — confirmar si existe o elegir sustituto.
- [ ] Automatizar pipeline diario (GitHub Action o cron local).
- [ ] Incorporar IBKR MCP para ejecución de señales en vivo.
