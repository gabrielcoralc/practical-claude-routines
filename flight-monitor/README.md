# Flight Monitor

Cloud Routine que monitorea diariamente precios de vuelos round-trip desde un
origen configurable y detecta **ofertas reales** comparando contra benchmarks
fijos y un histórico acumulado.

## Qué hace

1. Lee `vuelos/benchmarks.json` (umbrales por ruta) y `vuelos/historico.csv`.
2. Consulta el MCP de Kiwi.com para cada ruta configurada.
3. Calcula nivel de alerta combinando 4 señales: vs benchmark, vs media histórica,
   vs mínimo histórico, tendencia 7d.
4. Genera un reporte en `vuelos/reportes/YYYY-MM-DD.md` con bloque JSON
   estructurado al final (parseado luego por el workflow de Telegram).
5. Hace append de los precios del día al CSV.
6. Commitea todo con mensaje `vuelos: snapshot YYYY-MM-DD (<N> alertas)`.

## Niveles de alerta

| Nivel | Puntos | Significado |
|---|---|---|
| `[CHOLLO]` | 4+ | Precio anormalmente bajo, varias señales coinciden |
| `[OFERTA FUERTE]` | 3 | Buen precio confirmado por dos fuentes |
| `[OFERTA]` | 2 | Por debajo de benchmark "buen precio" |
| (sin alerta) | 0–1 | Precio normal o alto |

## Requisitos

- **MCP**: Kiwi.com (configurar en la routine para que tenga acceso a la
  herramienta `search-flight`).
- **Secrets** del repo target en GitHub:
  - `TELEGRAM_BOT_TOKEN` — bot de Telegram (vía @BotFather).
  - `TELEGRAM_CHAT_ID` — chat al que enviar notificaciones.

## Cómo configurar tu instancia

1. **Crea un repo privado** (ej. `mi-flight-monitor`) y copia esta carpeta:
   ```
   prompt.md
   vuelos/benchmarks.example.json   → renómbralo a benchmarks.json
   .github/workflows/auto-merge-routine.yml
   .github/workflows/notify-telegram.yml
   ```
   Crea también `vuelos/historico.csv` con solo el header:
   ```
   fecha_consulta,origen,destino,precio_cop,aerolinea,paradas,fecha_salida,fecha_regreso,dias_anticipacion,duracion_estancia,grupo
   ```

2. **Edita `benchmarks.json`** con tus rutas reales. Cada entrada:
   ```json
   "ORIGEN-DESTINO": { "grupo": "A|B|C", "tipico": ..., "buen_precio": ..., "oferta_fuerte": ... }
   ```
   Los grupos definen ventanas de anticipación y estancia (ver el bloque
   `ventanas` en el archivo).

3. **Edita `prompt.md`** ajustando los grupos de destinos a los tuyos. Las
   rutas de ejemplo apuntan a Pasto/Bogotá → MAD/MIA/EZE/Tokio; cámbialas.

4. **Configura los secretos** del repo en GitHub Settings → Secrets and
   variables → Actions.

5. **Crea la routine** en https://claude.ai/code/routines:
   - Pega el contenido de `prompt.md` como prompt.
   - Apunta al repo privado.
   - Schedule: `0 14 * * *` (9am Colombia, ajusta a tu zona).

## Workflows incluidos

### `auto-merge-routine.yml`

Mergea automáticamente las ramas `claude/**` que crea la routine a `main`.
Sin esto, cada ejecución queda aislada en su rama y `historico.csv` no
acumula nada. Salta el merge si la rama toca `prompt.md` o `README.md`
(cambios que merecen revisión manual). Tras mergear, borra la rama.

### `notify-telegram.yml`

Cuando se pushea un nuevo reporte a `vuelos/reportes/**.md`, parsea el
bloque JSON estructurado del final del reporte y envía un mensaje a
Telegram con las alertas del día (formato HTML, tope 8 alertas).

## Ajustes comunes

- **Cambiar moneda**: edita `prompt.md` (sección "Configuración") y la
  función `fmt_cop` en `notify-telegram.yml`.
- **Cambiar canal de notificación**: reemplaza `notify-telegram.yml` con
  uno que use Slack, Discord, o email. La estructura del bloque JSON al
  final del reporte está pensada para ser parseada por cualquier consumer.
- **Más o menos rutas**: edita `benchmarks.json` y los grupos en `prompt.md`.

## Limitaciones conocidas

- Kiwi.com a veces devuelve solo el tramo de ida — el prompt incluye una
  regla de validación pero puede colarse algún resultado erróneo.
- El histórico tarda ~30 días en ser estadísticamente útil para señales de
  desviación. Antes de eso, las alertas dependen casi solo de benchmarks.
- Los benchmarks son fijos en JSON; idealmente se ajustarían tras 60+ días
  con datos reales.
