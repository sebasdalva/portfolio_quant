# Portfolio Balanceado — Proyecto Cuantitativo (v3)

> Documento maestro. Leer primero al retomar. Estado: notebooks research + returns_risk operativos.
> Marcadores: [OK] decidido · [ ] pendiente · [!] a verificar

---

## 0. Estado del proyecto

| Area | Estado |
|---|---|
| Universo + tags | [OK] config/universe.csv |
| Fuente de datos | [OK] Yahoo (HTTP directo), agnostico |
| Almacenamiento | [OK] un CSV por ticker, OHLCV crudo + div + splits, append-only |
| Costos/tasas | [OK] config/costs_rates.csv |
| Infra Nivel 1 | [OK] carpeta Drive + Colab manual diario |
| Infra Nivel 2 (futuro) | [ ] repo Git privado + Claude Code + GitHub Action |
| Backtesting | [OK] quantstats + skfolio HRP (notebooks/returns_risk.ipynb) |
| Senales | [OK] regla ≤2 indicadores sobre observaciones distintas |
| Codigo fetch/adjust/signals | [OK] research.ipynb — migrado de Colab a local: paths relativos (`find_repo_root()`), kernel `.venv`, sin `drive.mount()` |
| Codigo returns/risk/backtest | [OK] returns_risk.ipynb (nuevo) |
| Ticker intl-trend (RSIT) | [!] no figura en la familia Return Stacked actual |

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

## 7. Infraestructura — dos niveles

- NIVEL 1 (ahora): carpeta Google Drive `portfolio_quant` con docs, configs y notebooks. Correr a mano a diario en Colab.
- NIVEL 2 (futuro): Claude Code + repo Git privado + GitHub Action (cron). IBKR por MCP.

## 8. Estructura de la carpeta (Nivel 1, Drive)

```
portfolio_quant/
├── README.md
├── portfolio_quant_project.md   # este doc — LEER PRIMERO
├── config/
│   ├── universe.csv
│   ├── costs_rates.csv
│   └── rules.yaml
├── data/prices/<TICKER>.csv     # lo crea research.ipynb
├── vendor/gauss314_skills/      # lo clona research.ipynb
└── notebooks/
    ├── research.ipynb            # baja/actualiza datos, senales
    ├── returns_risk.ipynb        # retornos, riesgo, backtest  ← NUEVO
    ├── tearsheet_equal_weight.html  # output quantstats
    └── tearsheet_momentum.html      # output quantstats
```

## 9. Flujo de trabajo (notebooks)

### research.ipynb — correr primero
1. Montar Drive.
2. Clonar/actualizar vendor/gauss314_skills.
3. Actualizar precios (append del cierre o backfill si CSV nuevo).
4. Calcular senales de momentum.

### returns_risk.ipynb — analisis
Depende de que research.ipynb haya bajado los datos. Celdas en orden:
1. Setup + mount Drive.
2. `%pip install -q quantstats skfolio`
3. Cargar CSVs y calcular adjusted close (funcion `load_adjusted`).
4. Estadisticas descriptivas por activo (quantstats).
5. Correlacion full + rolling 90d.
6. Portfolio EW (proxy bloques moviles) vs SPY.
7. Tearsheet HTML (quantstats) — se guarda en notebooks/.
8. Backtest con filtro momentum MA90 gradual 1/6 (`run_momentum_backtest`).
9. Curva de equity: Momentum overlay vs EW vs SPY.
10. Tabla de metricas comparadas.
11. Optimizacion HRP con skfolio.
12. Tabla final: Momentum(EW) vs HRP vs EW vs SPY.
13. Drawdown comparado.
14. Sharpe rolling 252d.
15. Atribucion anual por activo y por portfolio.
16. Guardar tearsheet momentum HTML.

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

Leer: este .md -> config/universe.csv -> config/rules.yaml -> notebooks/research.ipynb -> notebooks/returns_risk.ipynb.

Ruta de aprendizaje: (1) return stacking, (2) overlay de momentum 1/6, (3) almacenamiento crudo vs ajustado, (4) HRP/ERC con skfolio, (5) walk-forward CV, (6) pipeline diario.

## 13. Log de trades — formato

```
## YYYY-MM-DD — Rebalanceo [N]
Senales: RSSB v/x · RSST v/x · GLD v/x · SLV v/x
Trades: | Accion | Instrumento | Qty | Precio | Motivo |
Hedge fractions: | Asset | Hedge % |
Nota: (override, evento macro)
```

## 14. Proximos pasos sugeridos

- [ ] Correr returns_risk.ipynb y validar que los CSVs cargan bien.
- [ ] Revisar tearsheet_momentum.html: comparar Max DD del overlay vs benchmark.
- [ ] Experimentar con ERC (Equal Risk Contribution) en skfolio como alternativa a HRP.
- [ ] Agregar walk-forward CV al backtest (gauss314 skill).
- [ ] Resolver ticker RSIT — confirmar si existe o elegir sustituto.
- [ ] Incorporar bloques fijos (TIP, HYG/EMB, PFF) con precio diario para atribucion completa.
