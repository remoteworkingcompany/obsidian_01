---
title: Arquitectura de memoria
tags: [memoria, sistema]
---

# Arquitectura de memoria

Estructura de la carpeta `memory/`. Ver [[MEMORY]] para el índice curado.

## Carpetas

| Carpeta | Contenido | Retención |
|---|---|---|
| `daily/` | Cápsulas de contexto por sesión (`YYYY-MM-DD.md`) | 30 días, luego archivar |
| `decisions/` | Decisiones técnicas y arquitectónicas (`YYYY-MM-DD-titulo.md`) | Permanente |
| `lessons/` | Errores, patrones repetidos, aprendizajes | Permanente — nunca borrar |
| `archive/` | Logs diarios archivados tras 30 días | Indefinido |

## Reglas

- `daily/` es efímero: se escribe al cerrar sesión, se archiva en auditoría semanal
- `decisions/` es permanente: si se toma una decisión técnica importante, va aquí
- `lessons/` nunca se borra: si el mismo error ocurre dos veces, se convierte en regla en `AGENTS.md`
- `MEMORY.md` es el destilado: solo sube lo que es estable y de alta señal
