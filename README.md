# portfolio_quant — Nivel 1 (Drive + Colab manual)

Carpeta de trabajo del portafolio balanceado. Se opera **una vez al dia** a mano, post-cierre US, desde Colab.

## Que hay aca
- `portfolio_quant_project.md` — documento maestro (filosofia, reglas, decisiones, metodo). Es la fuente de verdad; se actualiza a medida que avanzamos.
- `config/universe.csv` — universo de tickers con tags (referencia vs ejecucion, structure, return_stacked, etc.). Agregar un ETF = una fila.
- `config/costs_rates.csv` — costos y tasas IBKR, mantenido a mano.
- `config/rules.yaml` — parametros de senales y rebalanceo (pocos, anti-overfitting).
- `data/prices/<TICKER>.csv` — precios OHLCV crudos (los crea el notebook).
- `vendor/gauss314_skills/` — copia de las skills de gauss314 (la clona el notebook).
- `notebooks/research.ipynb` — notebook de arranque (montar Drive, bajar precios, probar codigo).

## Como empezar
1. Abrir `notebooks/research.ipynb` con Google Colaboratory.
2. Correr las celdas en orden: montar Drive -> crear carpetas -> clonar skills -> smoke test de Yahoo -> cargar universe.csv.
3. A partir de ahi vamos sumando celdas (adjust -> signals -> backtest).

## Niveles
- **Nivel 1 (ahora):** todo en esta carpeta, manual, para entender el funcionamiento.
- **Nivel 2 (futuro):** proyectos independientes de la accion diaria con Claude Code + repo Git privado + GitHub Action.

## Notas
- Yahoo viene con 15 min de delay y endpoints no oficiales: bien para cierres diarios, pero el universo es agnostico a la fuente para poder cambiar de proveedor sin perder los datos guardados.
- Para optimizacion de pesos (HRP/ERC) usamos `skfolio` o `Riskfolio-Lib` (gauss314 todavia no trae portfolio optimization).
