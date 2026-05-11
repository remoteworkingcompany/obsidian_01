# MEMORY

Memoria curada. Máximo 2KB. Detalles técnicos completos en `~/.openclaw/workspace/openclaw/configuracion.md`.

## Puntos de entrada por temática

Workspace base: `~/.openclaw/workspace/`. Ante cualquier pregunta sobre contenido del vault, identificar temática y leer el nodo de entrada antes de responder.

| Temática | Nodo de entrada |
|---|---|
| Setup OpenClaw | `~/.openclaw/workspace/openclaw/openclaw.md` |
| Vault / Obsidian | `~/.openclaw/workspace/obsidian/configuracion.md` |
| Lista de la compra | `~/.openclaw/workspace/Lista de la compra.md` |
| Briefing del día | `~/.openclaw/workspace/briefings/YYYY-MM-DD.md` |
| Programación | `~/.openclaw/workspace/Programacion/` |
| IA | `~/.openclaw/workspace/IA/` |

## Stack

- Equipo: ASUS Ascent GX10 (NVIDIA GB10 Spark, 128GB UMA, ARM64) — DGX OS 7.5.0
- Modelo activo: `nemotron-omni` vía vLLM puerto 8001
- Embeddings: `bge-m3` vía vLLM puerto 8004
- Workspace OpenClaw: `~/.openclaw/workspace/` → symlink a `/home/dgltc/obsidian_vault/obsidian_windows/`

## Preferencias estables

- Idioma: castellano siempre
- Tono: directo, sin biblias, un paso cada vez
- NO bind mounts en k3s — usar rsync/lsyncd
- Acceso a UIs locales: SSH tunnel preferido sobre exposición LAN
- Reglas de inferencia y economía de contexto: ver `openclaw/configuracion.md`
