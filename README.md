# Practical Claude Code Routines

Ejemplos prácticos de **Cloud Routines** de Claude Code para automatizar tareas
recurrentes: monitoreo de precios, análisis diario de mercado, tracking de
features, etc. Cada carpeta es un caso de uso autocontenido con su prompt,
workflows de GitHub Actions y notas de configuración.

## ¿Qué es una Cloud Routine?

Una Cloud Routine es un agente de Claude Code (claude.ai/code/routines) que
corre automáticamente en un schedule (típicamente diario), ejecuta un prompt,
y commitea sus resultados a un repo de GitHub que tú defines.

Configuración mínima en la UI de claude.ai:

- **Prompt**: el texto que define qué hace la routine.
- **Repo target**: dónde commitea (necesita acceso de escritura).
- **Schedule**: cron (ej. todos los días a las 9am).

## Anatomía típica de una routine

```
mi-routine/
├── prompt.md                       # Documentación del prompt (la routine NO lo lee, se pega en la UI)
├── data/                           # CSV append-only, JSON de config, etc.
├── reportes/                       # Output diario en markdown
└── .github/workflows/
    ├── auto-merge-routine.yml      # Mergea automáticamente las ramas claude/** a main
    └── notify-telegram.yml         # Notifica el resultado a Telegram (u otro canal)
```

## Lecciones aprendidas

### 1. La routine crea una rama nueva en cada ejecución

Por seguridad, las Cloud Routines no pushean a `main` directamente. Cada
ejecución crea una rama tipo `claude/exciting-volta-XXX` y commitea ahí.

**Síntoma:** ves notificaciones diarias pero `main` nunca se actualiza, y los
CSVs append-only no acumulan histórico (cada rama parte de main fresco).

**Fix:** un workflow de GitHub Actions que detecte push a `claude/**` y haga
merge a main automáticamente. Ver [`flight-monitor/.github/workflows/auto-merge-routine.yml`](flight-monitor/.github/workflows/auto-merge-routine.yml).

### 2. El prompt vive en la UI, no en el repo

El `prompt.md` de cada routine es solo documentación. La routine ejecuta con
su copia interna del prompt (la que pegaste en claude.ai). Cuando edites el
prompt, recuerda copiarlo de vuelta a la UI.

### 3. CSV append-only para histórico

Para análisis vs histórico (medias móviles, comparaciones temporales), un CSV
append-only es lo más simple. La routine lee el CSV completo, calcula
estadísticas, y agrega su fila del día al final.

### 4. Notificaciones desacopladas vía workflow

En lugar de que la routine notifique directamente, deja que un workflow lo
haga al detectar el push del reporte. Beneficios: separación de
responsabilidades, retry fácil, secretos viven en GitHub.

## Routines en este catálogo

| Routine | Estado | Descripción |
|---|---|---|
| [flight-monitor](flight-monitor/) | Funcional | Monitoreo diario de precios de vuelos vía MCP de Kiwi.com con detección de ofertas vs benchmarks e histórico. |
| [stocks-watch](stocks-watch/) | Funcional | Análisis diario de un portafolio de acciones y crypto con recomendaciones (vender/comprar/observar) basadas en stop-loss, take-profit, señales técnicas y sentimiento de noticias (yfinance + Python). |
| [claude-changelog](claude-changelog/) | Planeado | Tracking diario de novedades de Claude Code (release notes, blog, GitHub releases) para mantenerse al día con nuevas features. |

## Cómo usar este catálogo

1. Mira el README de la routine que te interese.
2. Copia el `prompt.md` y los workflows a tu propio repo (privado si quieres
   guardar histórico personal).
3. Crea la routine en claude.ai/code/routines apuntando a tu repo.
4. Configura los secretos necesarios (`TELEGRAM_BOT_TOKEN`, etc.) en
   GitHub Settings → Secrets.
