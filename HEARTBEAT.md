# HEARTBEAT

Tareas proactivas que el agente revisa periódicamente. Marca con `[x]` lo completado.

## Mantenimiento de memoria (cada 3-7 días)
- [ ] Revisar archivos en `memory/` de los últimos 7 días
- [ ] Destilar insights clave en `MEMORY.md` (mantenerla corta y de alta señal)
- [ ] Eliminar entradas de `MEMORY.md` que ya no apliquen
- [ ] Archivar contenido extenso a `second-brain/` si crece demasiado

## Resumen semanal (cada domingo)
- [ ] Generar nota `memory/weekly-YYYY-WW.md` con:
  - Decisiones tomadas esta semana
  - Cosas aprendidas
  - Tareas completadas y pendientes
  - Patrones detectados (preguntas repetidas, problemas similares)

## Limpieza del vault (cada 2 semanas)
- [ ] Detectar notas huérfanas (sin links entrantes ni salientes)
- [ ] Detectar wikilinks rotos `[[X]]` que apunten a archivos inexistentes
- [ ] Sugerir merge de notas duplicadas o casi-duplicadas

## Recordatorios (cada vez que se invoque)
- [ ] Buscar `- [ ]` (checkboxes pendientes) en notas del vault
- [ ] Listar las que llevan >7 días sin tocar
- [ ] Preguntar al usuario si alguna sigue siendo relevante o se puede archivar

## Reglas
- NO ejecutar destrucción sin confirmación del usuario.
- Ante duda sobre relevancia de una memoria, mantenerla.
- Si se detecta info contradictoria entre `MEMORY.md` y archivos recientes, asumir que los recientes son verdad y proponer actualización.
