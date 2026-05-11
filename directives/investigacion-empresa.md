---
title: Workflow — Investigación de empresa
tags: [directiva, workflow, investigacion, empresas]
updated: 2026-05-11
---

# Workflow: Investigación de empresa

SOP para investigar una empresa y mantenerla actualizada. Ver [[analisis/empresas/README]] para estructura de archivos.

## Trigger

- **Estándar**: *"investiga [empresa]"*, *"añade [empresa]"*, *"busca info sobre [empresa]"*
- **Con URL semilla**: *"haz el setup de [empresa] a partir de: [url]"*, *"investiga [empresa], empieza por [url]"*
- **Con campos prioritarios**: el usuario puede añadir *"necesito especialmente: [campos]"* en cualquier variante

---

## Fase 1 — Investigación inicial (on-demand)

### Paso 1 — Identificación y modo de arranque

**Si el usuario proporciona una URL semilla:**
1. Hacer `web_fetch` de la URL semilla
2. Extraer de ella todo lo que sea posible: nombre legal, sector, país, datos básicos, links relevantes internos
3. Usar los links encontrados como punto de partida para el paso 2 (no buscar desde cero en SearXNG — expandir desde lo que ya hay)
4. Marcar la URL semilla como fuente `[verificada]` en `fuentes.md`

**Si no hay URL semilla:**
Confirmar con el usuario antes de buscar:
- Nombre exacto / nombre legal
- País y sector si se conoce
- Contexto: ¿para qué se investiga?

**En ambos casos:**
- Si el usuario especificó campos prioritarios, anotarlos → asegurarse de cubrirlos antes de pasar al paso siguiente
- Crear directorio: `~/.openclaw/workspace/analisis/empresas/{nombre-empresa}/`

### Paso 2 — Descubrimiento de fuentes

Buscar con SearXNG en este orden:

1. **Web corporativa**: página oficial, sección de sostenibilidad/ESG, sala de prensa, informes anuales
2. **Informes de sostenibilidad**: GRI, ESRS, SASB, TCFD, CDP — buscar PDFs publicados
3. **Datos financieros**: si es cotizada → CNMV, SEC, bolsa correspondiente. Si no → noticias financieras, Orbis/SABI si disponible
4. **Ratings ESG públicos**: CDP score, Sustainalytics (resúmenes públicos), MSCI ESG (datos públicos)
5. **Noticias relevantes**: últimos 12 meses — sostenibilidad, litigios, sanciones, cambios de dirección, M&A

### Paso 3 — Validación de fuentes

Por cada fuente encontrada, evaluar:
- **Primaria** (emitida por la propia empresa) vs **secundaria** (tercero)
- **Fecha**: ¿es reciente? ¿el informe es del último ejercicio?
- **Fiabilidad**: ¿fuente oficial, regulatoria, o medio con track record?

Descartar fuentes no verificables. Marcar las dudosas como `[pendiente verificar]`.

### Paso 4 — Extracción y estructuración

Crear `perfil.md` siguiendo el template en `~/.openclaw/workspace/analisis/templates/empresa-perfil.md`.

Rellenar todas las secciones con la información encontrada. Para cada dato, indicar la fuente entre paréntesis.

Si hay información suficiente sobre ESG:
→ Crear también `perfil_esg.md` desde `~/.openclaw/workspace/analisis/templates/empresa-perfil-esg.md`
→ Wikilink bidireccional entre `perfil.md` y `perfil_esg.md`

### Paso 5 — Documentos

Descargar a `docs/` los documentos más relevantes (informes de sostenibilidad, memorias anuales). Nombrarlos: `YYYY-titulo-corto.pdf`.

Referenciar cada documento descargado en `fuentes.md`.

### Paso 6 — Fuentes y seguimiento

Crear `fuentes.md` con todas las fuentes validadas, su tipo, fecha de última comprobación y frecuencia de actualización esperada.

Crear `seguimiento.md` con la config base para fase 2 (monitorización periódica).

### Paso 7 — Actualizar índice

Añadir la empresa a `~/.openclaw/workspace/analisis/empresas/README.md` con: nombre, sector, fecha de alta, contexto de investigación.

---

## Añadir un nuevo perfil especializado

Cuando el usuario pide profundizar en un tema concreto de una empresa ya investigada:

1. Crear `perfil_{tema}.md` en el directorio de la empresa
2. Wikilink desde `perfil.md` en la sección correspondiente
3. Usar SearXNG para complementar con fuentes específicas del tema
4. Actualizar `fuentes.md` con las nuevas fuentes

Temas frecuentes: `perfil_esg`, `perfil_financiero`, `perfil_legal`, `perfil_competencia`

---

## Actualización de perfil existente

Cuando el usuario pide actualizar información de una empresa ya en el vault:

1. Leer `perfil.md` y `fuentes.md` actuales
2. Comprobar fuentes activas en `seguimiento.md`
3. Buscar novedades desde la `ultima_actualizacion`
4. Actualizar secciones afectadas en `perfil.md` con fecha de actualización
5. Añadir entrada en `## Historial de cambios` de `perfil.md`
6. Si hay nuevos documentos → descargar a `docs/`

---

## Fase 2 — Monitorización periódica (cron)

Ver `seguimiento.md` de cada empresa. El HEARTBEAT lee estos archivos y comprueba cambios en las fuentes activas. Solo alerta si hay novedades relevantes.

Configuración en `seguimiento.md`:
- Frecuencia: diaria / semanal / mensual
- Fuentes a monitorizar (subset de `fuentes.md`)
- Criterios de "novedad relevante"
- Último check y resultado
