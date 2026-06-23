# Portfolio Strategies — Registro de Experimentos

**Propósito:** Registro de estrategias probadas en `signals.ipynb` y evaluadas en `backtest.ipynb`.  
Sigue las recomendaciones de López de Prado (2018) para evitar overfitting por múltiples comparaciones.  
Antes de agregar una estrategia, leer la sección de "Disciplina de backtesting" al final.

---

## Estrategias activas (en uso o en evaluación)

### STRAT-001 — Sistema Return Stacking con Hedge Gradual MA

**Descripción:** Portfolio de return stacking (RSSB, RSST, GLD, SLV, TIP, HYG, PFF) con filtro de momentum gradual. Cuando el precio cae por debajo de la media móvil, se hedgea 1/6 de la posición cada N días hasta cubrir el 100%. El hedge preserva el overlay de trend/bonds sin liquidar los ETFs.

**Parámetros:**
- MA length: 15 días
- Step size: 1/6 por rebalanceo
- Rebalanceo equity: cada 14 días
- Rebalanceo commodity: cada 30 días
- Hedge equity: short SPY (RSST), short VT (RSSB)
- Hedge commodity: venta directa (GLD, SLV)
- Colateral: BOXX (proxy histórico: SHY)
- Pesos: iguales por bloque (~1/7 cada uno)

**Archivo de señal:** `data/signals/daily_returns_STRAT001.csv`

**Período IS (in-sample):** _completar al testar_  
**Período OOS (out-of-sample):** _completar al testar_  
**Fecha de primer test:** _completar_  
**Número de trials acumulados:** 1

| Métrica | IS | OOS |
|---------|-----|-----|
| CAGR | — | — |
| Vol anual | — | — |
| Sharpe | — | — |
| Sortino | — | — |
| Max DD | — | — |
| Calmar | — | — |
| Win rate diario | — | — |

**Notas libres:**  
Sistema base del portfolio real. Parámetros derivados de la lógica del sistema, no de optimización sobre datos históricos. Punto de partida para variantes.

---

### STRAT-002 — 60/40 SPY/AGG (Benchmark)

**Descripción:** Portfolio clásico 60% equity (SPY) / 40% bonos (AGG). Rebalanceo mensual. Sin señal de momentum.

**Parámetros:**
- Pesos: SPY 60%, AGG 40%
- Rebalanceo: cada 21 días
- Sin hedge

**Archivo de señal:** `data/signals/daily_returns_STRAT002.csv`

**Período IS:** —  
**Período OOS:** —  
**Fecha de primer test:** _completar_  
**Número de trials acumulados:** 1

| Métrica | IS | OOS |
|---------|-----|-----|
| CAGR | — | — |
| Vol anual | — | — |
| Sharpe | — | — |
| Sortino | — | — |
| Max DD | — | — |
| Calmar | — | — |
| Win rate diario | — | — |

**Notas libres:**  
Benchmark de referencia. No se modifica. Sirve como baseline para calcular el valor agregado de las otras estrategias.

---

### STRAT-003 — Trend Following puro (SPY con MA binaria)

**Descripción:** SPY con señal binaria: 100% SPY si precio > MA, 100% SHY/BOXX si precio < MA. Sin gradualidad. Posición de tamaño completo o cero.

**Parámetros:**
- MA length: 15 días (por comparar con STRAT-001)
- Señal: binaria (0 o 1)
- Colateral cuando fuera: SHY/BOXX
- Rebalanceo: diario (se revisa la señal cada día)

**Archivo de señal:** `data/signals/daily_returns_STRAT003.csv`

**Período IS:** —  
**Período OOS:** —  
**Fecha de primer test:** _completar_  
**Número de trials acumulados:** 1

| Métrica | IS | OOS |
|---------|-----|-----|
| CAGR | — | — |
| Vol anual | — | — |
| Sharpe | — | — |
| Sortino | — | — |
| Max DD | — | — |
| Calmar | — | — |
| Win rate diario | — | — |

**Notas libres:**  
Caso de referencia para entender el valor de la gradualidad (1/6) vs la señal binaria pura.

---

## Estrategias en cola (ideas para probar)

- STRAT-004: MA variable (10d, 25d, 60d) — sensibilidad del sistema al parámetro MA
- STRAT-005: Pesos proporcionales al inverso de la volatilidad (risk parity simple)
- STRAT-006: Sistema con hedge en Kelly fraccional en vez de 1/6 fijo

---

## Disciplina de backtesting (López de Prado)

Reglas a seguir antes de agregar cualquier estrategia a esta lista:

1. **Definir IS y OOS antes de correr.** Nunca mirar OOS hasta tener los parámetros fijados en IS.
2. **Registrar el número de trials.** Cada vez que se modifica un parámetro y se re-testea, +1 al contador. El Sharpe IS pierde significado con muchos trials.
3. **Deflated Sharpe.** Si el número de trials supera 5 sobre la misma ventana de datos, calcular DSR antes de declarar la estrategia válida.
4. **Separación temporal mínima.** IS y OOS deben ser períodos distintos y no solapados. OOS debe ser al menos 20% del total.
5. **No re-usar OOS.** Una vez evaluado en OOS, ese período queda "contaminado". Si se modifica la estrategia, hay que buscar un nuevo OOS.
6. **Documentar por qué cambia un parámetro.** Si se cambia la MA de 15 a 25 días, registrar la hipótesis antes de ver el resultado.

---

## Archivos relacionados

| Archivo | Rol |
|---------|-----|
| `notebooks/signals.ipynb` | Genera señales y `daily_returns_XXXX.csv` |
| `notebooks/backtest.ipynb` | Consume señales y produce métricas de performance |
| `config/rules.yaml` | Parámetros operativos del sistema en producción |
| `config/universe.csv` | Universo de tickers activos |
