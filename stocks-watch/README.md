# Stocks Watch *(planeado)*

Cloud Routine que analiza diariamente un portafolio de acciones compradas y
genera recomendaciones de **comprar / vender / mantener** basadas en
límites configurables (stop-loss, take-profit) y señales de mercado.

> **Estado:** todavía no implementada. Este README documenta la propuesta
> de diseño antes de construirla.

## Qué haría

1. Lee `portafolio.json` (tickers comprados, cantidades, precio de compra,
   límites de venta y compra adicional).
2. Consulta precios actuales (fuente por decidir — ver "Decisiones pendientes").
3. Lee `historico.csv` con precios diarios acumulados.
4. Para cada posición, evalúa:
   - **Stop-loss tocado**: precio actual ≤ límite_venta_baja → recomendar **VENDER**.
   - **Take-profit tocado**: precio actual ≥ límite_venta_alta → recomendar **VENDER**.
   - **Oportunidad de compra adicional**: precio ≤ límite_compra → recomendar **COMPRAR MÁS**.
   - **Tendencia anómala**: caída/subida >X% vs media 30d → señalar.
5. Genera reporte en `reportes/YYYY-MM-DD.md` con secciones:
   - Resumen P&L total
   - Acciones a tomar HOY (con razón)
   - Tabla de posiciones (precio, %, vs límites)
   - Bloque JSON para Telegram
6. Append a `historico.csv`.
7. Commit + notificación.

## Decisiones pendientes antes de implementar

### Fuente de precios

| Opción | Pros | Contras |
|---|---|---|
| `yfinance` (Python) | Gratis, sin API key, cobertura global | A veces tira errores de scraping, requiere Python en el runtime |
| Alpha Vantage | API estable, gratis con key | Solo 25 req/día en free tier |
| Finnhub | 60 req/min gratis | Cobertura más limitada en mercados emergentes |
| MCP financiero | Integración nativa con Claude | Disponibilidad por confirmar |

**Sugerencia:** empezar con `yfinance` ejecutado vía Bash desde la routine
(la routine puede correr `python -c "import yfinance..."` para obtener
precios). Si Kiwi.com funciona como MCP en flight-monitor, ver si existe
algo equivalente para stocks.

### Mercado objetivo

¿Acciones US (NYSE/NASDAQ), Colombia (BVC), o ambas? Define el formato de
ticker y el horario de la routine (NYSE cierra 4pm ET = 3pm Colombia).

### Granularidad de límites

Por ticker fijo (`AAPL: vender si <150`) o porcentaje sobre precio de
compra (`vender si -10%`). Lo segundo escala mejor y se autoajusta.

## Estructura propuesta

```
mi-stocks-watch/
├── prompt.md
├── portafolio.json              # tus posiciones + límites (PRIVADO)
├── historico.csv                # precios diarios append-only
├── reportes/YYYY-MM-DD.md
└── .github/workflows/
    ├── auto-merge-routine.yml   # mismo patrón que flight-monitor
    └── notify-telegram.yml      # adaptado al formato de reporte
```

## Próximos pasos

1. Decidir fuente de precios y mercado objetivo.
2. Diseñar `portafolio.json` (formato exacto).
3. Escribir `prompt.md` siguiendo la estructura de flight-monitor.
4. Adaptar `notify-telegram.yml` al nuevo bloque JSON (tickers, P&L,
   recomendaciones).
5. Crear repo privado, configurar routine, primera corrida en seco.
