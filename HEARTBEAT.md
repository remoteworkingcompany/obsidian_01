# HEARTBEAT

Sistema de mantenimiento proactivo del workspace y del agente.

**Regla de oro: silencio = todo bien.** Solo interrumpir si algo necesita acción del usuario. Errar siempre hacia el silencio, no hacia el ruido.

## Cadencia

### Cada 30 minutos — health check silencioso
- Gateway OpenClaw activo (`systemctl --user is-active openclaw-gateway`)?
- vLLM respondiendo (`curl -s localhost:8001/v1/models`)?
- SearXNG ok (`curl -s localhost:8888/search?q=test&format=json`)?
- Cron jobs ejecutándose según lo esperado?

Si todo ok → log silencioso, no notificar.
Si algo falla → notificar con: qué falló, diagnóstico probable, comando de fix sugerido.
Si el fix es obvio, no destructivo y reversible → aplicarlo y notificar después.

### Cada mañana — briefing condicional
Solo si hay algo que requiera atención:
- ¿Hay tarea de alta prioridad en `openclaw/pendiente.md` con deadline hoy?
- ¿Algún error del día anterior sin resolver en logs?
- ¿El briefing `briefings/YYYY-MM-DD.md` está generado?

Si nada relevante → silencio.

### Cada lunes — auditoría semanal
Ejecutar silenciosamente. Al terminar, proponer **una sola mejora** al usuario.

**Memoria:**
- [ ] `MEMORY.md` sigue por debajo de 2KB? Si no, compactar.
- [ ] Logs en `memory/daily/` de más de 30 días? Archivar en `memory/archive/` o eliminar si no hay nada relevante.
- [ ] Hay decisiones de la semana no registradas en `memory/decisions/`?
- [ ] `memory/lessons/` tiene entradas nuevas que deberían subir a `MEMORY.md`?

**Configuración:**
- [ ] Algún archivo de sistema (SOUL, AGENTS, MEMORY) que haya crecido innecesariamente?
- [ ] Alguna skill que no se use hace más de 2 semanas?
- [ ] Wikilinks rotos en el vault (`[[X]]` que no apuntan a ningún archivo)?

**Patrones:**
- [ ] El usuario ha repetido la misma pregunta más de 2 veces este mes? Documentar shortcut en `AGENTS.md`.
- [ ] Algún error recurrente que no esté en `openclaw/problemas.md`?

**Mejora de la semana:**
Basada en los patrones observados, proponer UNA mejora concreta al usuario en formato:
> "He detectado [patrón]. Propuesta: [acción específica]. ¿Lo implemento?"

### Primer lunes de mes — auditoría mensual
- [ ] Tareas en `openclaw/pendiente.md` con más de 30 días → preguntar si siguen vigentes
- [ ] Bugs en `openclaw/problemas.md` marcados como activos pero resueltos informalmente?
- [ ] `memory/decisions/` — hay decisiones que deberían subir a `MEMORY.md` como hechos estables?
- [ ] Proponer una mejora de infraestructura basada en lo observado el mes anterior

## Reglas

- Usar el modelo más ligero disponible para checks rutinarios
- Nunca destruir sin confirmación explícita del usuario
- Ante duda sobre relevancia de una memoria: mantenerla
- Si algo contradice una decisión anterior registrada: alertar antes de actuar
- Los logs de heartbeat van a `memory/daily/YYYY-MM-DD.md`, no al chat
