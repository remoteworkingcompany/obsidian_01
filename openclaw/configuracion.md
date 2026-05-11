---
title: Configuración OpenClaw
tags: [setup, openclaw, vllm, nemotron, spark, configuracion]
updated: 2026-05-11
---

# Configuración OpenClaw

> Estado: funcional con ajustes pendientes (ver [[pendiente]])

Documento de referencia del setup actual de OpenClaw. Ver [[openclaw]] para índice general, [[obsidian/configuracion]] para infraestructura del vault, [[MEMORY]] para reglas de inferencia, [[AGENTS]] para operación del agente.

## Hardware

- **Equipo**: ASUS Ascent GX10 (NVIDIA GB10 Spark)
- **Chip**: GB10 Grace Blackwell, sm_121
- **RAM**: 128GB LPDDR5x unificada (UMA)
- **Bandwidth**: 273 GB/s
- **Arquitectura**: ARM64
- **OS**: Ubuntu 24.04 (DGX OS 7.5.0)
- **Driver NVIDIA**: 580.142
- **CUDA**: 13.0/13.1

## Red

- **Hostname Linux**: `gx10-5ec9`
- **Hostname Tailscale**: `gx10` (alias)
- **IP Tailscale**: `100.117.181.92`
- **IP LAN**: `192.168.68.58`

## Inferencia

### vLLM — modelo principal (puerto 8001)

- **Contenedor**: `vllm-nemotron-omni`
- **Imagen**: `vllm/vllm-openai:v0.20.0`
- **Modelo**: `nvidia/Nemotron-3-Nano-Omni-30B-A3B-Reasoning-FP8`
- **Served name**: `nemotron-omni`
- **Context window**: 131072 tokens
- **Max output**: 8192 tokens
- **Capacidades**: texto + imagen + audio + video, reasoning
- **Flags clave**:
  - `--trust-remote-code`
  - `--max-model-len 131072`
  - `--gpu-memory-utilization 0.65`
  - `--kv-cache-dtype fp8`
  - `--enable-prefix-caching`
  - `--reasoning-parser nemotron_v3`
  - `--enable-auto-tool-choice --tool-call-parser qwen3_coder`

### vLLM — embeddings (puerto 8004)

- **Contenedor**: `vllm-bge-m3`
- **Modelo**: `BAAI/bge-m3`

### Ollama (puerto 11434)

- Instalado, sin modelos activos. Auxiliar puntual.

## OpenClaw

- **Versión**: 2026.5.7 (eeef486)
- **Instalación**: nativa en `~/.openclaw/`
- **Node**: 22.22.0 (propio, no global)
- **Provider**: vLLM apuntando a `http://127.0.0.1:8001/v1`
- **Model**: `nemotron-omni`
- **Gateway**: systemd user service, puerto 18789, bind loopback
- **Auth**: token en `~/.openclaw/config` (no versionar)
- **Workspace**: `~/.openclaw/workspace` → symlink a vault Obsidian

### Servicios systemd user

| Servicio | Estado |
|---|---|
| `openclaw-gateway` | running |
| `lsyncd` | deshabilitado (no necesario) |
| `openclaw-node` | desinstalado (causaba bug `dir.list not allowed`) |

### Channels

- **Discord**: bot `dgltc_openclaw_bot` (ID `1502620911587823687`)
  - App ID y token en `channels.discord` dentro de config de OpenClaw
  - Intents activos: Message Content, Server Members, Presence

### Skills

**Bundled**: `browser-automation`, `discord`, `healthcheck`, `node-connect`, `skill-creator`, `taskflow`, `taskflow-inbox-triage`, `tmux`, `weather`

**Custom workspace**: `obsidian-openclaw-memory` (en `~/.openclaw/workspace/skills/obsidian-memory`)

### Web search

- **Provider**: SearXNG (`http://localhost:8888`)
- **Container**: `searxng` (`searxng/searxng:latest`)
- **Config**: `~/searxng/docker-compose.yml`

## Email

- **MTA**: msmtp + msmtp-mta
- **Config**: `~/.msmtprc` (chmod 600)
- **SMTP**: Gmail (`smtp.gmail.com:587`)
- **Cuenta**: `openclawdgltc@gmail.com`
- **Auth**: App password de Google (en `~/.msmtprc`, no versionar)

## Scripts custom

- `~/.openclaw/scripts/morning-briefing.sh` (monolito, deprecated)
- `~/.openclaw/scripts/briefing-parts/`:
  - `00-prensa.sh`
  - `01-acciones.sh`
  - `02-cripto.sh`
  - `03-viento.sh` (Windguru, falla a veces)
  - `04-tiempo-mareas.sh`

## Acceso remoto

- **Desde Windows**: SSH tunnel `ssh -fN -L 18789:localhost:18789 dgltc@gx10`
- **Dashboard**: `http://localhost:18789/` (requiere token)

## Bugs conocidos y workarounds

Ver [[problemas]].

## Pendiente

Ver [[pendiente]].

## Histórico

Ver [[historico]].
