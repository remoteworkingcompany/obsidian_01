---
title: Comandos útiles OpenClaw
tags: [setup, comandos, cheatsheet]
created: 2026-05-10
---

# Comandos útiles

Cheatsheet de operaciones frecuentes. Ver [[openclaw]] para índice general, [[configuracion]] para puertos, container names y paths de referencia.

## OpenClaw

### Estado y diagnóstico
```bash
openclaw status
openclaw status --all                # Reporte completo (redacta secretos)
openclaw --version
openclaw skills list
openclaw skills check
openclaw skills info <name>
openclaw devices list
openclaw channels list
```

### Logs
```bash
openclaw logs --follow
openclaw logs --max-bytes 50000 2>&1 | tail -30
journalctl --user -u openclaw-gateway -f
journalctl --user -u openclaw-gateway -n 50 --no-pager
```

### Gateway
```bash
systemctl --user restart openclaw-gateway
systemctl --user status openclaw-gateway --no-pager
systemctl --user is-active openclaw-gateway
openclaw configure --section web                     # Reconfigurar web search
```

### Sesiones y chat
```bash
openclaw agent --agent main --session-id "test-$(date +%s)" \
  --message "tu pregunta" --thinking off
ls ~/.openclaw/agents/main/sessions/*.jsonl
```

### Discord pairing
```bash
openclaw pairing approve discord <code>
```

## vLLM

### Estado del modelo
```bash
curl -s localhost:8001/v1/models | python3 -m json.tool
curl -s localhost:8001/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{"model":"nemotron-omni","messages":[{"role":"user","content":"ping"}],"max_tokens":50}'
```

### Container management
```bash
docker ps | grep vllm
docker logs vllm-nemotron-omni --tail 50
docker stats vllm-nemotron-omni --no-stream
docker restart vllm-nemotron-omni
docker inspect vllm-nemotron-omni --format='{{.Config.Cmd}}'   # Ver flags actuales
```

### GPU
```bash
nvidia-smi
nvidia-smi --query-gpu=utilization.gpu,memory.used --format=csv,noheader
```

## SearXNG

```bash
cd ~/searxng && docker compose ps
cd ~/searxng && docker compose restart
curl -s "http://localhost:8888/search?q=test&format=json" -o /dev/null -w "HTTP %{http_code}\n"
docker logs searxng --tail 30
```

## Email (msmtp)

```bash
echo -e "Subject: test\n\nfunciona" | msmtp openclawdgltc@gmail.com
tail -20 ~/.msmtp.log
```

## Scripts de briefing

```bash
~/.openclaw/scripts/briefing-parts/00-prensa.sh
~/.openclaw/scripts/briefing-parts/01-acciones.sh
~/.openclaw/scripts/briefing-parts/02-cripto.sh
~/.openclaw/scripts/briefing-parts/03-viento.sh         # falla a veces
~/.openclaw/scripts/briefing-parts/04-tiempo-mareas.sh
ls -la ~/.openclaw/workspace/briefings/
cat ~/.openclaw/workspace/briefings/$(date +%Y-%m-%d).md
```

## Acceso remoto

### Desde Windows
```powershell
ssh -fN -L 18789:localhost:18789 dgltc@gx10
Test-NetConnection localhost -Port 18789
# Luego abrir http://localhost:18789/ en navegador con token
```

### Desde Mac
```bash
ssh -fN -L 18789:localhost:18789 dgltc@gx10
```

### Token de acceso
```bash
# El token está en ~/.openclaw/openclaw.json → gateway.auth.token
# No mostrarlo en logs ni documentos versionados
cat ~/.openclaw/openclaw.json | python3 -c "import json,sys; print(json.load(sys.stdin)['gateway']['auth']['token'])"
```

## Config OpenClaw

### Ver config completa
```bash
cat ~/.openclaw/openclaw.json | python3 -m json.tool
```

### Backup antes de editar
```bash
cp ~/.openclaw/openclaw.json ~/.openclaw/openclaw.json.$(date +%Y%m%d-%H%M%S).bak
```

### Cambiar maxTokens del modelo
```bash
python3 -c "
import json
p='/home/dgltc/.openclaw/openclaw.json'
c=json.load(open(p))
c['models']['providers']['vllm']['models'][0]['maxTokens']=4096
json.dump(c, open(p,'w'), indent=2)
"
systemctl --user restart openclaw-gateway
```

## Vault Obsidian

```bash
ls ~/.openclaw/workspace/
ls ~/.openclaw/workspace/briefings/
ls ~/.openclaw/workspace/skills/
readlink ~/.openclaw/workspace                       # Confirmar symlink
```

## Sistema

```bash
free -h
df -h /home
hostnamectl
tailscale status
```

## Reinicios típicos en cascada

Cuando algo va mal, reiniciar en este orden:

```bash
# 1. Gateway (lo más frecuente)
systemctl --user restart openclaw-gateway
sleep 5 && openclaw status

# 2. vLLM (si el modelo no responde)
docker restart vllm-nemotron-omni
sleep 30 && curl -s localhost:8001/v1/models | grep -i nemotron

# 3. SearXNG (si búsqueda falla)
cd ~/searxng && docker compose restart

# 4. Todo (último recurso)
systemctl --user restart openclaw-gateway
docker restart vllm-nemotron-omni vllm-bge-m3 searxng
```
