# Stocks Watch

Cloud Routine que analiza diariamente un portafolio de acciones y crypto,
y emite recomendaciones (`VENDER`, `VENDER PARCIAL`, `COMPRAR MÁS`,
`OBSERVAR`, `MANTENER`) basadas en límites configurables (stop-loss,
take-profit) más señales técnicas (tendencia, drawdown, volatilidad) y
sentimiento de noticias recientes.

## Qué hace

1. Lee `data/portafolio.json` (tus posiciones + cost basis + límites) y
   `data/historico.csv` (precios diarios acumulados).
2. Vía `pip install yfinance && python` consulta:
   - Precio actual y 60 días de histórico por ticker.
   - Top 3 noticias por ticker (`yf.Ticker(symbol).news`).
3. Calcula métricas: precio actual, % vs compra, valor USD, media 20d,
   drawdown 30d, volatilidad 30d anualizada, tendencia 30d.
4. Evalúa recomendación combinando:
   - **Triggers de límites** (stop-loss, take-profit, buy-more) por ticker.
   - **Señales técnicas** (caída sostenida, momentum positivo).
   - **Sentimiento de noticias** (positivo / negativo / neutral) que
     refuerza o atenúa la recomendación.
5. Genera reporte en `reportes/YYYY-MM-DD.md` con resumen, acciones
   recomendadas, tabla completa, noticias destacadas y bloque JSON
   estructurado para Telegram.
6. Append a `historico.csv`.
7. Commit + notificación a Telegram (auto-merge a main vía workflow).

## Ejecución en el entorno cloud

La routine corre `pip install yfinance pandas numpy` al inicio (el entorno
de Cloud Routines tiene Python disponible). Las llamadas a `yfinance` son
gratis y sin API key. Una corrida típica consulta 10-15 tickers en menos
de 2 minutos.

## Niveles de recomendación

| Nivel | Trigger principal |
|---|---|
| `VENDER` | Stop-loss tocado o take-profit completo |
| `VENDER PARCIAL` | Take-profit + señal de momentum positivo (mantener fracción) |
| `COMPRAR MÁS` | Precio bajó por debajo del `buy_more_pct` configurado vs compra |
| `OBSERVAR` | Señal técnica (drawdown sostenido, tendencia negativa) sin trigger duro |
| `MANTENER` | Sin señales accionables |

Las noticias actúan como **modificador**: 2+ negativas refuerzan recomendaciones
negativas; 2+ positivas pueden convertir un `VENDER` por take-profit en
`VENDER PARCIAL` (mantener fracción por momentum).

## Requisitos

- **Secrets** del repo target en GitHub:
  - `TELEGRAM_BOT_TOKEN` — bot de Telegram (vía @BotFather).
  - `TELEGRAM_CHAT_ID` — chat al que enviar notificaciones.
- **MCP**: ninguno requerido. yfinance es Python puro.
- **Sin API keys externas** para precios ni noticias (yfinance las trae todas).

## Cómo configurar tu instancia

1. **Crea un repo privado** (ej. `mi-stocks-watch`) y copia esta carpeta:
   ```
   prompt.md
   data/portafolio.example.json   → renómbralo a data/portafolio.json
   data/historico.csv
   .github/workflows/auto-merge-routine.yml
   .github/workflows/notify-telegram.yml
   ```

2. **Edita `data/portafolio.json`** con tus posiciones reales:
   - Cada ticker: `yf_symbol` (ej. `META`, `BTC-USD` con guión para crypto),
     `shares`, `precio_compra` (cost basis tuyo), `tipo` (`stock` o `crypto`).
   - `config_default` aplica a todas las posiciones `stock`; cada ticker
     puede sobrescribir con sus propios `stop_loss_pct`, `take_profit_pct`,
     `buy_more_pct` (o dejar `null` para usar el default).
   - `config_default_crypto` aplica a posiciones `tipo: "crypto"` con
     thresholds más amplios por volatilidad.

3. **Edita `prompt.md`** si quieres ajustar la lógica (umbrales técnicos,
   formato del reporte, etc.).

4. **Configura los secretos** en GitHub Settings → Secrets and variables → Actions.

5. **Crea la routine** en https://claude.ai/code/routines:
   - Pega el contenido de `prompt.md` como prompt.
   - Apunta al repo privado.
   - Schedule: `0 22 * * 1-5` (5pm ET, después del cierre de mercado US,
     ajusta a tu zona). Si tienes crypto que vale la pena revisar fines de
     semana, usa `0 22 * * *`.

## Workflows incluidos

### `auto-merge-routine.yml`
Idéntico al de flight-monitor. Mergea automáticamente las ramas
`claude/**` a main, salvo que toquen `prompt.md` o `README.md`. Tras
mergear, borra la rama. Tiene reintentos con rebase para race conditions.

### `notify-telegram.yml`
Detecta push a `reportes/**.md`, parsea el bloque JSON estructurado del
final del reporte y envía a Telegram un mensaje con:
- Valor total del portafolio + cambio diario.
- Conteos de recomendaciones por nivel.
- Hasta 8 acciones recomendadas con ticker, precio, %, drawdown, sentimiento
  de noticias y razón principal.
- Link al reporte completo.

## Ajustes comunes

- **Cambiar moneda de display**: edita `fmt_usd` en `notify-telegram.yml` y
  la sección "Configuración" del prompt. yfinance siempre devuelve precios
  en la moneda de listado del ticker.
- **Más métricas técnicas**: agrega RSI, MACD, etc. al paso 3 del prompt.
  yfinance + pandas/numpy ya están en el entorno.
- **Posiciones sin cost basis**: si `precio_compra` es `null`, la routine
  omite señales de stop-loss/take-profit y solo reporta señales técnicas.
  Útil para posiciones recién abiertas.

## Limitaciones conocidas

- yfinance ocasionalmente tira errores de scraping (Yahoo cambia su HTML).
  La routine los maneja por ticker — si uno falla, los demás continúan.
- Las noticias de `yf.Ticker.news` son las que Yahoo Finance agrega; no
  cubre todas las fuentes. Para cobertura más amplia, agregar WebSearch
  por ticker.
- El sentimiento de noticias lo clasifica el agente leyendo titulares —
  cualitativo, no cuantitativo. Suficiente para alertas direccionales.
- La routine no considera dividendos en el cálculo de `pct_vs_compra`;
  para posiciones que pagan dividendos altos, el % real de retorno es
  mayor al reportado.

## Notas para crypto

Tickers crypto en yfinance usan formato `SIMBOLO-USD`:
- BTC → `BTC-USD`
- ETH → `ETH-USD`
- XRP → `XRP-USD`

Marca esas posiciones con `"tipo": "crypto"` en `portafolio.json` para
que apliquen los thresholds más amplios de `config_default_crypto`.
