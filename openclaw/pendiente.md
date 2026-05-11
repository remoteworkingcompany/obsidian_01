---
title: Pendiente OpenClaw
tags: [setup, todo, pendiente]
created: 2026-05-10
---

# Pendiente

Ver [[openclaw]] para índice general, [[configuracion]] para estado actual y [[problemas]] para bugs activos.

## Alta prioridad

- [x] **Afinar AGENTS.md**: el agente sigue confundiéndose con queries triviales; faltan ejemplos de qué archivo abrir para cada tipo de pregunta

## Media prioridad

- [ ] **HEARTBEAT.md ejecutable**: configurar cron interno de OpenClaw para revisar heartbeat periódicamente sin bucle
- [ ] **QMD semantic search**: instalar y configurar QMD para búsqueda semántica en el vault (ver `skills/obsidian-memory/SKILL.md`)
- [ ] **Plugins Obsidian Windows**: instalar Dataview y Templater en Obsidian Desktop Windows
- [ ] **Daily note template**: configurar Templater con plantilla para `memory/YYYY-MM-DD.md`
- [ ] **Plugin allowlist OpenClaw**: configurar `plugins.allow` para evitar warning recurrente en logs

## Baja prioridad

- [ ] **Resumen semanal automático**: cron domingos → `memory/weekly-YYYY-WW.md`
- [ ] **Limpieza vault**: detectar wikilinks rotos, notas huérfanas
- [ ] **Recordatorios de tareas**: buscar `- [ ]` antiguos en vault y proponer archivar
- [ ] **Cripto/acciones con APIs reales**: CoinGecko, Yahoo Finance en lugar de web_search
- [ ] **Mareas con API**: API de mareas de Puertos del Estado o AEMET

## Investigar

- [ ] Por qué Discord pierde mensajes después de respuestas largas (issue #4864)
- [ ] Por qué logs del gateway a veces no muestran actividad Discord
- [ ] Si conviene mover orquestación a un workflow más estable (n8n, Temporal, etc.)
