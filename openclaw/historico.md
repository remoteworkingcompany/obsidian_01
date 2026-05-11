---
title: Histórico de cambios OpenClaw
tags: [setup, historico, changelog]
created: 2026-05-10
---

# Histórico de cambios

> Cronología del setup. Lo más reciente arriba. Ver [[openclaw]] para índice general, [[configuracion]] para estado actual y [[problemas]] para bugs resueltos con detalle.

## 2026-05-11 — Fix inyección de contexto en sesiones

- Detectado que el agente ignoraba el vault y buscaba en web para preguntas sobre contenido local
- Causa raíz 1: `MEMORY.md` superaba el límite de 5000 chars de `bootstrapMaxChars` → se truncaba antes de llegar a la tabla de puntos de entrada
- Causa raíz 2: sin instrucción explícita de arranque de sesión, el agente no leía `MEMORY.md` activamente
- Solución 1: reducir `MEMORY.md` a <2KB (1337 bytes), moviendo detalles técnicos a `openclaw/configuracion.md`
- Solución 2: tabla de puntos de entrada movida al principio de `MEMORY.md`
- Solución 3: añadida instrucción explícita en `SOUL.md` → leer `MEMORY.md` al arrancar cada sesión
- Nota: `bootstrap` no es una clave válida en `agents.list` de `openclaw.json` — no intentar añadirla

## 2026-05-10 — Recuperación tras reinicio

- Detectado bug `dir.list not allowed` que bloqueaba al agente al listar workspace
- Causa: `openclaw-node` service no soporta `dir.list` en Linux (solo iOS/Android)
- Solución: `openclaw node uninstall`, dejar solo el gateway
- Añadido a `AGENTS.md` lista de archivos importantes con nombres exactos

## 2026-05-09 (tarde) — Setup OpenClaw segundo cerebro

- Clonado skill `obsidian-openclaw-memory` en `~/.openclaw/workspace/skills/obsidian-memory`
- Skill detectado como `✓ ready` tras restart del gateway
- Creados archivos base del sistema de memoria: `USER.md`, `SOUL.md`, `AGENTS.md`, `MEMORY.md`, `HEARTBEAT.md`
- Instalado msmtp + msmtp-mta para envío de email vía Gmail con App Password
- Configurado SearXNG en Docker tras detectar que DuckDuckGo daba timeout y Brave eliminó tier gratis
- Activado formato JSON en SearXNG (`~/searxng/config/settings.yml`)
- Cambiado provider de búsqueda a SearXNG vía `openclaw configure --section web`
- Creado script `morning-briefing.sh` para briefing diario por email
- Dividido en `briefing-parts/` (00–04) tras detectar que el monolito saturaba contexto
- Subido `max-model-len` de vLLM a 131072 (era 40960 inicialmente)
- Conectado bot Discord `dgltc_openclaw_bot` (ID `1502620911587823687`)
  - Habilitados Privileged Gateway Intents (Message Content, Server Members, Presence)
  - Pairing aprobado: `openclaw pairing approve discord 7PEGUBQR`
- Fix bug visión Nemotron: `models.providers.vllm.models[0].input` → `["text", "image"]`
- Cambiado `bind` del gateway de `tailnet` a `loopback` para que SSH tunnel funcione

## 2026-05-09 (mañana) — Migración NemoClaw → OpenClaw

- Desinstalado NemoClaw (uninstaller oficial roto, limpieza manual):
  - `nemoclaw my-assistant destroy --yes` + `nemoclaw nemo destroy --yes`
  - Eliminados containers Docker, volumes, binarios y configs de NemoClaw
  - `rm -rf ~/.nemoclaw ~/.config/openshell ~/.config/nemoclaw`
- Eliminado lsyncd (ya no necesario sin sandbox k3s)
- Instalado OpenClaw nativo: `curl -fsSL https://openclaw.ai/install-cli.sh | bash`
- Onboarding eligiendo: gateway local, provider vLLM, model `nemotron-omni`, puerto 18789, auth token, SearXNG
- Systemd lingering ON, gateway service instalado

## 2026-05-09 (madrugada) — Lucha con NemoClaw

- Detectada limitación NemoClaw: k3s + sandbox no acepta bind mounts al vault Obsidian
- Intentos fallidos con bind mount del PVC → pod crashea
- Workaround temporal con lsyncd (host → sandbox cada 2s vía inotify)
- Detectado bug NemoClaw multimodal: catálogo cerrado de modelos, no soporta vision custom
- Workaround: editar `openclaw.json` dentro del sandbox para forzar `input: ["text", "image"]`
- Frustración acumulada → decisión de migrar a OpenClaw standalone

## Pre-2026-05-09 — Configuración base

- Setup inicial NemoClaw con Ollama + nemotron-3-super:120b
- Cambio a vLLM con Nemotron-3-Nano-Omni-30B-A3B-Reasoning-FP8
- Configurado SSH tunnel desde Mac/Windows para acceso al dashboard
- Flags vLLM optimizados: `--kv-cache-dtype fp8`, `--gpu-memory-utilization 0.6` (luego 0.65), `--enable-prefix-caching`, `--reasoning-parser nemotron_v3`, `--enable-auto-tool-choice --tool-call-parser qwen3_coder`
