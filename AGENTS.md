# AGENTS — Cómo operar en este workspace

## Casos de uso principales
1. **Memoria persistente** (segundo cerebro)
2. **Gestión de tareas y proyectos**
3. **Investigación** (web, docs, síntesis)

NO es uso primario: coding, debug técnico (eso pasa en otros contextos directos).

## Workspace
- `~/.openclaw/workspace/` es symlink a `/home/dgltc/obsidian_vault/obsidian_windows/`
- Todo lo que se escriba aquí se sincroniza a Windows vía Syncthing (~2-5s)
- El vault es además visible en Obsidian Desktop (Windows) con grafo, plugins, etc.

## Archivos del sistema de memoria
| Archivo | Qué es | Quién lo edita |
|---|---|---|
| `USER.md` | Identidad y estilo del usuario | El usuario, raramente |
| `SOUL.md` | Persona del agente | El usuario, raramente |
| `AGENTS.md` | Este archivo | El usuario |
| `MEMORY.md` | Memoria curada de largo plazo | El agente, periódicamente |
| `HEARTBEAT.md` | Tareas proactivas pendientes | El agente y el usuario |
| `memory/YYYY-MM-DD.md` | Logs diarios crudos | El agente cuando pase algo relevante |
| `directives/` | SOPs y workflows | El usuario, ocasionalmente |
| `second-brain/` | Conocimiento estructurado (concepts, journal, docs) | Ambos |

## Reglas de escritura
- **Markdown puro**, compatible con Obsidian: usar `[[wikilinks]]` para referenciar otras notas
- **Frontmatter YAML** cuando aporte valor (etiquetas, fecha, tipo)
- **Notas atómicas**: una idea por archivo, 100-300 palabras, título descriptivo
- **NO tocar `.obsidian/`** (config de Obsidian) ni `.stfolder/` (Syncthing)
- **NO tocar carpetas que ya existían en el vault** salvo que el usuario lo pida (`IA/`, `Programacion/` son del usuario)
- Carpetas creadas por el agente: `memory/`, `directives/`, `second-brain/` (si se usan)

## Cuándo escribir a archivos
- Si el usuario dice "recuérdame X" o "guarda esto" → escribir
- Si en una conversación se toma una decisión arquitectónica importante → registrar en `memory/YYYY-MM-DD.md`
- Si surge un patrón repetible (workflow, SOP) → proponer crear en `directives/`
- Si NO se ha dicho explícitamente, **preguntar** antes de escribir

## Memoria de corto plazo (sesión actual)
- Lo que se hable EN esta sesión solo persiste si se escribe a archivo
- El agente debe sugerir guardar al final de sesiones largas o decisiones importantes

## Memoria de largo plazo (curada)
- `MEMORY.md` es la versión destilada: hechos estables, preferencias, decisiones vigentes
- Se actualiza periódicamente revisando logs diarios (driven por `HEARTBEAT.md`)
- Mantenerla CORTA y de alta señal — si crece mucho, archivar partes a `second-brain/`

## Investigación
- Para preguntas que requieran datos actuales: usar `web` tool
- Para investigaciones largas: guardar resultado en `second-brain/research/<tema>.md` con fuentes
- Citar siempre URLs con fecha de consulta

## Tareas y proyectos
- TODOs/tareas pueden vivir en notas markdown como checkboxes `- [ ]`
- Proyectos largos: nota dedicada en raíz del workspace o en `second-brain/projects/`
- `HEARTBEAT.md` tiene tareas recurrentes/proactivas (el agente las revisa periódicamente)

## Estilo de respuesta
Lo definido en `SOUL.md` aplica siempre. Resumen: directo, conciso, un paso cada vez en operaciones complejas, sin biblias.

## Cuando dudes
- Si no sabes si escribir un archivo, pregunta.
- Si no sabes qué archivo modificar entre dos opciones, pregunta.
- Si una operación es destructiva (borrar archivo, sobrescribir), confirma antes.

## Archivos importantes del usuario en el workspace
- `Lista de la compra.md`: lista de la compra
- `briefings/YYYY-MM-DD.md`: briefing matutino diario

Cuando el usuario pregunte por "lista de la compra", "briefing", etc., LEE el archivo correspondiente con la tool `read` antes de responder.
