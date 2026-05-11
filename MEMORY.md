# MEMORY

Memoria curada de largo plazo. Hechos estables. Si algo aquí deja de ser cierto, actualizar o eliminar.

## Hardware
- **Equipo principal**: ASUS Ascent GX10 (NVIDIA GB10 Spark, sm_121, 128GB UMA, 273GB/s, ARM64)
- **Hostname Tailscale**: `gx10-5ec9` (IP `100.117.181.92`)
- **Nodo auxiliar**: Mac Studio M2 Max (64GB, ARM64) con MLX para modelos pequeños/medianos y embeddings
- **Acceso remoto**: Tailscale mesh VPN entre todos los dispositivos
- **OS Spark**: DGX OS 7.5.0, driver 580.142, CUDA 13.0/13.1

## Inferencia activa
- **Modelo principal**: `nvidia/Nemotron-3-Nano-Omni-30B-A3B-Reasoning-FP8` (multimodal: texto+imagen+audio+video)
- **Servido como**: `nemotron-omni` en vLLM (`vllm/vllm-openai:v0.20.0`) en puerto 8001
- **Embeddings**: `BAAI/bge-m3` en vLLM puerto 8004
- **Container nombres**: `vllm-nemotron-omni`, `vllm-bge-m3`
- **Ollama**: instalado, auxiliar puntual (puerto 11434)

## Stack del usuario
- Vault Obsidian: `/home/dgltc/obsidian_vault/obsidian_windows/` (sincronizado a Windows vía Syncthing)
- Carpetas pre-existentes en vault: `IA/`, `Programacion/` (NO tocar salvo orden explícita)
- Workspace OpenClaw: `~/.openclaw/workspace/` → symlink al vault

## Reglas de inferencia (Spark / GB10)
- **Cuantización preferida**: NVFP4 > FP8 > BF16. Evitar AWQ/GPTQ-INT4 (calidad irregular en sm_121)
- **vLLM** es motor por defecto en Spark para producción. Ollama solo auxiliar puntual
- **MLX** es motor preferido en Mac (NO Ollama en Mac, MLX rinde 1.5-2.5× mejor)
- **Modelos <14B y single-stream** → Mac. **Modelos grandes / VLM pesado / concurrencia** → Spark
- **Embeddings, reranking, ASR** → Mac (siempre que esté disponible)

## Economía de contexto en GB10
- Prefill rapidísimo (~1800-2200 tok/s), decode lento (~30-70 tok/s)
- 1 token de output cuesta ~50× lo que 1 token de input
- **Regla**: más input bien construido > menos output ambiguo
- Para tareas determinísticas (clasificación, extracción): siempre `enable_thinking: false` o `/no_think`

## Setup OpenClaw
- Versión: 2026.5.7, instalación nativa en `~/.openclaw/`
- Gateway: systemd user service, bind loopback, puerto 18789, auth por token
- Acceso remoto: SSH tunnel desde Windows (`ssh -fN -L 18789:localhost:18789 dgltc@gx10`)
- Provider configurado: vLLM apuntando a `http://127.0.0.1:8001/v1` con `nemotron-omni`
- Workspace = vault Obsidian, así Windows ve todo lo que escribe el agente

## Bug conocido OpenClaw + vLLM
- OpenClaw tiene whitelist hardcoded de modelos con visión. `nemotron-omni` NO está incluido.
- Workaround aplicado: editar `openclaw.json` para forzar `input: ["text", "image"]` en el provider vllm
- Si el agente "no ve imágenes" después de actualizar OpenClaw, revisar este punto

## Casos de uso del Spark
- Pipelines ESG/CSRD: extracción y análisis de documentos (ESRS, reports, knowledge graphs IRO)
- Stack típico: Marker (PDFs) + VLM (tablas complejas) + pgvector/Qdrant + Neo4j/Memgraph + e5-large
- Patrón input/output: 10-20k tokens input → 200-800 tokens JSON estructurado output

## Preferencias estables
- Idioma: castellano siempre
- Tono: directo, sin biblias, un paso cada vez
- NO usar bind mounts en cluster k3s (propagación rprivate, no funciona). Usar rsync/lsyncd
- Para acceso a UIs locales desde otros nodos: SSH tunnel preferido sobre exposición LAN
