# Claude Code Changelog Tracker *(planeado)*

Cloud Routine que revisa diariamente las novedades publicadas por Anthropic
sobre Claude Code y genera un reporte con las features, fixes y cambios
del día. Útil para mantenerse al día sin tener que monitorear manualmente
release notes, blog y GitHub releases.

> **Estado:** todavía no implementada. Este README documenta la propuesta
> de diseño.

## Qué haría

1. Lee `historico.json` (últimos N items reportados, para detectar novedades).
2. Consulta fuentes de novedades vía WebFetch:
   - **Release notes oficiales:** https://docs.claude.com/en/release-notes/claude-code
   - **GitHub releases:** https://github.com/anthropics/claude-code/releases
   - **Anthropic news / blog:** https://www.anthropic.com/news (filtrar
     posts con tag "Claude Code" o "Engineering")
   - *(Opcional)* Discord de Anthropic, foro de desarrolladores.
3. Diff contra el histórico para identificar items nuevos del día.
4. Para cada item nuevo:
   - Resumen en 2-3 líneas (qué es, para qué sirve, impacto práctico).
   - Categoría (`feature`, `fix`, `breaking`, `deprecation`, `docs`).
   - Link a la fuente original.
5. Genera reporte en `reportes/YYYY-MM-DD.md`. Si no hay novedades, reporte
   con "Sin novedades hoy" y commit normal (para mantener el ritmo).
6. Actualiza `historico.json` con los items nuevos.
7. Commit + notificación a Telegram (resumida — solo headlines + link al
   reporte completo).

## Decisiones pendientes antes de implementar

### Cómo evitar duplicados / falsos positivos de "novedad"

Las páginas de release notes a veces se editan retroactivamente (typos,
clarificaciones). Necesitamos hash o ID estable por item:

- GitHub releases tienen tag (`v1.x.y`) — ID natural.
- Release notes en docs.claude.com pueden no tener IDs — usar fecha + título
  como key, o hash del contenido del item.
- Blog posts tienen URL única.

**Sugerencia:** el `historico.json` guarda `{source, id_o_url, fecha_visto,
hash_contenido}`. Si reaparece con hash distinto, reportar como
"actualizado" en vez de "nuevo".

### Frecuencia y horario

Diario a la mañana (Colombia) parece suficiente — Anthropic publica en
horario US, así que las novedades del día anterior US ya están disponibles.

### Filtro de relevancia

¿Reportar TODO o solo lo relevante para el desarrollador típico?

- **Reportar todo:** más completo, pero ruido (cambios menores de docs).
- **Filtrar por categoría:** dejar al prompt clasificar y omitir `docs` triviales.

**Sugerencia:** reportar todo pero categorizado, así el resumen de Telegram
puede mostrar solo `feature` + `breaking` + `fix` y dejar `docs` y otros
en el reporte completo.

## Estructura propuesta

```
mi-claude-changelog/
├── prompt.md
├── historico.json               # items vistos, para diff
├── reportes/YYYY-MM-DD.md
└── .github/workflows/
    ├── auto-merge-routine.yml
    └── notify-telegram.yml
```

## Plantilla de reporte

```markdown
# Changelog Claude Code YYYY-MM-DD

## Resumen
N novedades hoy: X features, Y fixes, Z otros.

## Novedades

### [FEATURE] Título del item
- **Fuente:** docs.claude.com/release-notes
- **Resumen:** qué es, para qué sirve, cómo usarlo en 2-3 líneas.
- **Link:** https://...

### [FIX] ...

## Datos (no-render)
```json
{
  "fecha": "YYYY-MM-DD",
  "conteos": {"feature": 0, "fix": 0, "breaking": 0, "deprecation": 0, "docs": 0},
  "items": [...]
}
```

## Próximos pasos

1. Confirmar fuentes (¿algunas otras además de las 3 listadas?).
2. Decidir formato de `historico.json` (cómo identificar items únicos).
3. Escribir `prompt.md` con instrucciones claras de WebFetch + diff.
4. Adaptar `notify-telegram.yml` al nuevo bloque JSON.
5. Crear repo, configurar routine, primera corrida.
