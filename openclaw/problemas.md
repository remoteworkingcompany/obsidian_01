---
title: Problemas conocidos OpenClaw
tags: [setup, bugs, troubleshooting]
created: 2026-05-10
---

# Problemas conocidos y workarounds

Ver [[openclaw]] para índice general, [[configuracion]] para setup actual y [[historico]] para cronología de cambios.

## Activos


### Discord pierde mensajes en operaciones largas (>2 min)
- **Síntoma**: bot muestra "respondiendo" pero el mensaje nunca llega a Discord
- **Causa**: WS de Discord timeout, [bug conocido OpenClaw #4864](https://github.com/openclaw/openclaw/issues/4864)
- **Workaround**: para tareas largas, usar webchat o CLI directo en lugar de Discord
- **Estado**: sin fix upstream

### Logs del gateway no muestran mensajes Discord recibidos a veces
- **Síntoma**: bot responde pero `journalctl` no registra recepción
- **Causa**: bug de logging silencioso en OpenClaw 2026.5.7
- **Workaround**: usar `openclaw status --all` para ver actividad real
- **Estado**: no investigado

### Briefing matutino: Windguru no parseable
- **Síntoma**: script `03-viento.sh` se cuelga generando tabla de Windguru
- **Causa**: `web_fetch` sobre Windguru devuelve HTML JS-heavy inútil; el modelo se atasca
- **Workaround pendiente**: cambiar fuente (Windy, AEMET) o usar API real de Windguru
- **Estado**: pendiente (ver [[pendiente]])

### Briefing matutino monolito satura contexto
- **Síntoma**: script único genera 290 tool calls truncados y se atasca
- **Causa**: demasiadas secciones en un solo prompt
- **Workaround**: dividido en `briefing-parts/` (5 scripts pequeños)
- **Estado**: parcialmente resuelto

## Resueltos

### Agente ignora vault y busca en web para preguntas sobre contenido local
- **Síntoma**: "qué hay en la lista de la compra?" → el agente va a web search en lugar de leer el archivo
- **Causa 1**: `MEMORY.md` superaba el límite de 5000 chars (`bootstrapMaxChars`) y se truncaba antes de llegar a la tabla de puntos de entrada
- **Causa 2**: sin instrucción explícita de arranque, el agente no leía `MEMORY.md` activamente
- **Solución**: reducir `MEMORY.md` a <2KB, poner la tabla de puntos de entrada al principio, añadir instrucción de arranque en `SOUL.md`
- **Regla**: mantener `MEMORY.md` siempre por debajo de 2KB. Detalles técnicos van en `openclaw/configuracion.md`
- **Fecha**: 2026-05-11

### `dir.list not allowed for platform "linux"`
- **Síntoma**: agente intenta listar workspace con `node` tool y entra en bucle infinito
- **Causa**: `openclaw-node` intenta usar `dir.list` bloqueado en Linux
- **Solución**: `openclaw node uninstall`. El agente usa `bash` o `read` en su lugar
- **Fecha**: 2026-05-10

### `Origin not allowed` desde IP Tailscale
- **Síntoma**: dashboard inaccesible desde otra IP que no sea localhost
- **Solución**: cambiar `bind` a `loopback`, acceder vía SSH tunnel
- **Fecha**: 2026-05-09

### `control ui requires device identity (use HTTPS or localhost)`
- **Síntoma**: browser bloquea acceso a UI por HTTP plano sobre IP
- **Solución**: SSH tunnel `ssh -fN -L 18789:localhost:18789 dgltc@gx10`
- **Fecha**: 2026-05-09

### Discord 4014 missing privileged gateway intents
- **Síntoma**: bot conecta y se desconecta inmediatamente
- **Solución**: activar Message Content, Server Members y Presence Intent en Bot settings del Developer Portal
- **Fecha**: 2026-05-09

### Discord 401 Unauthorized
- **Síntoma**: bot no autentica
- **Causa**: usaba OAuth client secret en lugar de bot token
- **Solución**: regenerar bot token desde Developer Portal → Bot → Reset Token
- **Fecha**: 2026-05-09

### `Failed to resolve Discord application id`
- **Solución**: añadir `channels.discord.applicationId = "1502620911587823687"` en `openclaw.json`
- **Fecha**: 2026-05-09

### Nemotron-omni no procesa imágenes en OpenClaw
- **Causa**: `models.providers.vllm.models[0].input` registrado como `["text"]`
- **Solución**: cambiar a `["text", "image"]` en `openclaw.json`
- **Fecha**: 2026-05-09

### DuckDuckGo timeouts constantes
- **Solución**: migrar a SearXNG self-hosted en Docker
- **Fecha**: 2026-05-09

### Brave API ya no gratis
- **Causa**: Brave eliminó tier gratis en feb 2026
- **Solución**: SearXNG (sin key, ilimitado)
- **Fecha**: 2026-05-09

### Context overflow en briefings
- **Síntoma**: error `40961 tokens > 40960 max`
- **Solución**: subir `max-model-len` de vLLM a 131072
- **Fecha**: 2026-05-09

### NemoClaw bind mount no propaga al pod k3s
- **Causa**: Docker mount propagation `rprivate`, k3s no refresca PVC montados
- **Workaround**: rsync/lsyncd push host → PVC
- **Solución final**: migrar a OpenClaw standalone
- **Fecha**: 2026-05-09

### NemoClaw multimodal whitelist hardcoded
- **Causa**: catálogo cerrado de modelos con visión
- **Solución final**: migrar a OpenClaw standalone
- **Fecha**: 2026-05-08
