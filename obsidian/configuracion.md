---
title: Configuración Obsidian
tags: [obsidian, vault, infraestructura, configuracion]
updated: 2026-05-11
---

# Configuración Obsidian

Infraestructura del vault de Obsidian. El vault actúa como hub central: es simultáneamente el workspace de [[openclaw/configuracion]], la interfaz de gestión del sistema y el punto de acceso desde Windows a todo el stack Linux. Ver también [[MEMORY]] para hardware y [[AGENTS]] para reglas de operación del agente.

## Paths del vault

| Dispositivo | Path |
|---|---|
| Linux (GX10) | `/home/dgltc/obsidian_vault/obsidian_windows/` |
| Windows | `C:\Users\dgltc\obsidian_vault\obsidian_windows\` |
| OpenClaw workspace | `~/.openclaw/workspace/` → symlink al path Linux |

## Sincronización entre dispositivos

- **Motor**: Syncthing
- **Latencia**: ~2-5 segundos entre Linux y Windows
- **Dirección**: bidireccional — lo que escribe el agente en Linux aparece en Obsidian Desktop (Windows) en segundos
- **Carpetas excluidas de sync**: `.git/`, `.stfolder/`

## Control de versiones

- **Repo**: `git@github-remoteworkingcompany:remoteworkingcompany/obsidian_01.git`
- **Cuenta GitHub**: `remoteworkingcompany`
- **Clave SSH**: `~/.ssh/remoteworkingcompany`
- **Alias SSH**: `github-remoteworkingcompany` (en `~/.ssh/config` de Windows)
- **Rama principal**: `master`
- **Propósito**: backup y reconstrucción completa del workspace si hay pérdida de datos

## Obsidian como hub de acceso para OpenClaw

El vault no es solo almacenamiento — es la interfaz principal a través de la cual OpenClaw accede y gestiona:

- **Memoria del agente**: `MEMORY.md`, `memory/YYYY-MM-DD.md` (logs diarios)
- **Configuración del agente**: `AGENTS.md`, `IDENTITY.md`, `SOUL.md`, `USER.md`, `TOOLS.md`
- **Documentación del sistema**: `openclaw/`, `obsidian/` (estas carpetas)
- **Contenido del usuario**: `Programacion/`, `IA/`, `briefings/`, `Lista de la compra.md`
- **Tareas proactivas**: `HEARTBEAT.md`
- **Skills**: `skills/` (submodulos con capacidades externas)

Todo lo que el agente escribe en el vault es inmediatamente visible y editable desde Obsidian Desktop en Windows, creando un flujo bidireccional usuario ↔ agente.

## Plugins activos

- **Obsidian Kanban**: gestión de tareas en tablero visual

## Reglas de escritura del agente en el vault

- Markdown puro, compatible con Obsidian
- Frontmatter YAML cuando aporte valor
- `[[wikilinks]]` para referenciar otras notas y construir el grafo
- Notas atómicas: una idea por archivo, 100-300 palabras
- NO tocar `.obsidian/` ni `.stfolder/`
- NO tocar carpetas pre-existentes del usuario (`IA/`, `Programacion/`) salvo orden explícita
