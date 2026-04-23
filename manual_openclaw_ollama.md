# Manual Técnico: Stack IA Local — OpenClaw + Ollama
> Autohospedaje completo. Sin suscripciones. Privacidad total.

---

## Prerrequisitos del Sistema

| Componente | Mínimo | Recomendado |
|---|---|---|
| RAM | 16 GB | 32 GB+ |
| VRAM (GPU NVIDIA) | 6 GB | 12 GB+ |
| Disco para modelos | 50 GB | 200 GB+ (NVMe preferido) |
| OS | Ubuntu 22.04 LTS | Ubuntu 24.04 LTS |
| CPU | x86_64 con AVX2 | x86_64 con AVX512 |

```bash
# Verificar soporte AVX/AVX2 en CPU
grep -o 'avx[^ ]*' /proc/cpuinfo | sort -u
```

---

## 1. Preparación del Disco Nuevo

### 1.1 Identificar el disco

```bash
lsblk -o NAME,SIZE,TYPE,MOUNTPOINT,FSTYPE
# Identificar el disco sin montar, ej: /dev/sdb (sin particiones activas)

# Verificar que es el disco correcto ANTES de formatear
sudo fdisk -l /dev/sdb
```

> ⚠️ **CRÍTICO**: Confirmar el device name. Un error aquí destruye datos.

### 1.2 Particionar y formatear

```bash
# Crear tabla de particiones GPT y partición única
sudo parted /dev/sdb --script mklabel gpt
sudo parted /dev/sdb --script mkpart primary ext4 0% 100%

# Formatear con ext4 (etiqueta descriptiva para identificación futura)
sudo mkfs.ext4 -L ia_data /dev/sdb1

# Verificar
lsblk -f /dev/sdb
```

### 1.3 Montar y persistir en fstab

```bash
# Crear punto de montaje
sudo mkdir -p /mnt/ia_data

# Obtener UUID (no usar /dev/sdX, cambia entre reinicios)
sudo blkid /dev/sdb1
# Ejemplo de salida: UUID="a1b2c3d4-e5f6-..."

# Agregar a fstab
sudo nano /etc/fstab
```

Agregar al final de `/etc/fstab`:
```
UUID=a1b2c3d4-e5f6-xxxx-xxxx-xxxxxxxxxxxx  /mnt/ia_data  ext4  defaults,noatime  0  2
```

```bash
# Montar sin reiniciar y verificar
sudo mount -a
df -h /mnt/ia_data

# Crear subdirectorios del stack
sudo mkdir -p /mnt/ia_data/{ollama_models,openclaw_data,logs}
sudo chown -R $USER:$USER /mnt/ia_data
```

---

## 2. Dependencias y Drivers

### 2.1 Sistema base

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y curl wget git build-essential ca-certificates gnupg lsb-release
```

### 2.2 Opción A — Con GPU NVIDIA

```bash
# Verificar GPU detectada
lspci | grep -i nvidia

# Instalar drivers privativos (recomendado vía ubuntu-drivers)
sudo ubuntu-drivers autoinstall
# O instalación manual del driver específico:
# sudo apt install nvidia-driver-535

# Reiniciar obligatorio
sudo reboot

# Verificar post-reinicio
nvidia-smi
```

**Instalar NVIDIA Container Toolkit:**

```bash
# Agregar repositorio oficial NVIDIA
curl -fsSL https://nvidia.github.io/libnvidia-container/gpgkey \
  | sudo gpg --dearmor -o /usr/share/keyrings/nvidia-container-toolkit-keyring.gpg

curl -s -L https://nvidia.github.io/libnvidia-container/stable/deb/nvidia-container-toolkit.list \
  | sed 's#deb https://#deb [signed-by=/usr/share/keyrings/nvidia-container-toolkit-keyring.gpg] https://#g' \
  | sudo tee /etc/apt/sources.list.d/nvidia-container-toolkit.list

sudo apt update
sudo apt install -y nvidia-container-toolkit

# Configurar Docker runtime para NVIDIA
sudo nvidia-ctk runtime configure --runtime=docker
sudo systemctl restart docker

# Test rápido de validación
docker run --rm --gpus all nvidia/cuda:12.0-base-ubuntu22.04 nvidia-smi
```

### 2.3 Opción B — Solo CPU (sin NVIDIA)

```bash
# Verificar extensiones vectoriales disponibles
grep -c avx2 /proc/cpuinfo

# Ollama en modo CPU usa automáticamente AVX/AVX2 si están disponibles
# No requiere configuración adicional, pero los modelos correrán más lentos
# Recomendado: modelos de 7B o menos (llama3.2:3b, qwen2.5-coder:7b)
```

### 2.4 Instalar Docker y Docker Compose v2

```bash
# Remover instalaciones antiguas
sudo apt remove -y docker docker-engine docker.io containerd runc 2>/dev/null

# Instalar Docker Engine oficial
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg \
  | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg

echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] \
  https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" \
  | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

# Agregar usuario al grupo docker (evita usar sudo en cada comando)
sudo usermod -aG docker $USER
newgrp docker

# Verificar versiones
docker --version
docker compose version
```

---

## 3. Despliegue del Stack

### 3.1 Opción Docker — Compose Unificado

Crear estructura de proyecto:

```bash
mkdir -p ~/ia-local && cd ~/ia-local
```

Crear `docker-compose.yml`:

```yaml
# ~/ia-local/docker-compose.yml
version: "3.9"

networks:
  ia_net:
    driver: bridge

volumes:
  ollama_data:
    driver: local
    driver_opts:
      type: none
      o: bind
      device: /mnt/ia_data/ollama_models
  openclaw_data:
    driver: local
    driver_opts:
      type: none
      o: bind
      device: /mnt/ia_data/openclaw_data

services:

  ollama:
    image: ollama/ollama:latest
    container_name: ollama
    restart: unless-stopped
    networks:
      - ia_net
    ports:
      - "11434:11434"
    volumes:
      - ollama_data:/root/.ollama
    environment:
      - OLLAMA_HOST=0.0.0.0
      - OLLAMA_KEEP_ALIVE=24h
      - OLLAMA_MAX_LOADED_MODELS=2
    # --- Habilitar solo con GPU NVIDIA ---
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              count: all
              capabilities: [gpu]
    # --- Fin bloque GPU ---

  openclaw:
    image: ghcr.io/openclaw/openclaw:latest   # Ajustar tag según release actual
    container_name: openclaw
    restart: unless-stopped
    depends_on:
      - ollama
    networks:
      - ia_net
    ports:
      - "3000:3000"
    volumes:
      - openclaw_data:/app/data
    environment:
      - OLLAMA_BASE_URL=http://ollama:11434
      - WEBUI_SECRET_KEY=cambia_esto_por_una_clave_segura
      - DEFAULT_MODELS=llama3.2
    
  openjarvis:
    image: ghcr.io/openjarvis/openjarvis:latest   # Ver sección 5
    container_name: openjarvis
    restart: unless-stopped
    depends_on:
      - ollama
    networks:
      - ia_net
    ports:
      - "8080:8080"
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock   # Acceso OS para automatización
      - /mnt/ia_data/logs:/app/logs
    environment:
      - OLLAMA_URL=http://ollama:11434
```

> ℹ️ Si no tienes GPU, elimina el bloque `deploy.resources.reservations` del servicio `ollama`.

```bash
# Levantar el stack completo
docker compose up -d

# Verificar estado de contenedores
docker compose ps

# Logs en tiempo real
docker compose logs -f ollama
```

### 3.2 Opción Nativa (sin Docker)

**Instalar Ollama binario:**

```bash
curl -fsSL https://ollama.com/install.sh | sh

# Configurar OLLAMA_HOST para acceso en red local
sudo mkdir -p /etc/systemd/system/ollama.service.d
sudo tee /etc/systemd/system/ollama.service.d/override.conf <<EOF
[Service]
Environment="OLLAMA_HOST=0.0.0.0"
Environment="OLLAMA_MODELS=/mnt/ia_data/ollama_models"
Environment="OLLAMA_KEEP_ALIVE=24h"
EOF

sudo systemctl daemon-reload
sudo systemctl enable --now ollama
sudo systemctl status ollama
```

**Instalar OpenClaw nativo:**

```bash
# Prerrequisito: Node.js 20+
curl -fsSL https://deb.nodesource.com/setup_20.x | sudo -E bash -
sudo apt install -y nodejs

# Clonar y compilar
git clone https://github.com/openclaw/openclaw.git /opt/openclaw
cd /opt/openclaw
npm install
cp .env.example .env

# Editar .env
nano .env
# Líneas clave:
# OLLAMA_BASE_URL=http://localhost:11434
# WEBUI_SECRET_KEY=clave_segura_aqui
# DATA_DIR=/mnt/ia_data/openclaw_data

npm run build
npm start

# Opcional: convertir en servicio systemd
sudo tee /etc/systemd/system/openclaw.service <<EOF
[Unit]
Description=OpenClaw Web UI
After=network.target ollama.service

[Service]
Type=simple
User=$USER
WorkingDirectory=/opt/openclaw
ExecStart=/usr/bin/node server.js
Restart=on-failure
EnvironmentFile=/opt/openclaw/.env

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl enable --now openclaw
```

---

## 4. Configuración OpenClaw + Modelos Ollama

### 4.1 Pull de modelos recomendados

```bash
# Modelos con contexto 32k-128k — ajustar según VRAM disponible

# General purpose — contexto 128k, 2GB VRAM (cuantización Q4)
ollama pull llama3.2

# Coding — contexto 32k, 4-8GB VRAM
ollama pull qwen2.5-coder:7b
# Para GPUs con 12GB+:
ollama pull qwen2.5-coder:14b

# Embeddings (para RAG en OpenClaw)
ollama pull nomic-embed-text

# Con Docker: ejecutar dentro del contenedor
docker exec -it ollama ollama pull llama3.2
docker exec -it ollama ollama pull qwen2.5-coder:7b
docker exec -it ollama ollama pull nomic-embed-text

# Listar modelos descargados
ollama list
# o
docker exec -it ollama ollama list
```

### 4.2 Verificar API de Ollama

```bash
# Test directo del endpoint
curl http://localhost:11434/api/tags | python3 -m json.tool

# Test de inferencia
curl http://localhost:11434/api/generate \
  -H "Content-Type: application/json" \
  -d '{"model":"llama3.2","prompt":"Hola, responde en una línea","stream":false}'
```

### 4.3 Vincular OpenClaw al endpoint local

```bash
# Comando onboard (si OpenClaw lo soporta vía CLI)
openclaw onboard --ollama-url http://localhost:11434

# En Docker:
docker exec -it openclaw openclaw onboard --ollama-url http://ollama:11434
```

Alternativamente via UI web `http://localhost:3000`:
1. **Settings → Connections → Ollama**
2. URL: `http://ollama:11434` (Docker) o `http://localhost:11434` (nativo)
3. Save → Test Connection

---

## 5. Ecosistema y Extensiones

### 5.1 VSCode — Extensión Continue

```bash
# Instalar desde marketplace
code --install-extension Continue.continue
```

Configurar `~/.continue/config.json`:

```json
{
  "models": [
    {
      "title": "Qwen2.5 Coder (Local)",
      "provider": "ollama",
      "model": "qwen2.5-coder:7b",
      "apiBase": "http://localhost:11434"
    },
    {
      "title": "Llama 3.2 (Local)",
      "provider": "ollama",
      "model": "llama3.2",
      "apiBase": "http://localhost:11434"
    }
  ],
  "tabAutocompleteModel": {
    "title": "Qwen2.5 Coder Autocomplete",
    "provider": "ollama",
    "model": "qwen2.5-coder:7b",
    "apiBase": "http://localhost:11434"
  },
  "embeddingsProvider": {
    "provider": "ollama",
    "model": "nomic-embed-text",
    "apiBase": "http://localhost:11434"
  }
}
```

> Si Ollama corre en otra máquina de la red local, reemplazar `localhost` por la IP LAN, ej: `http://192.168.1.100:11434`.

### 5.2 VSCode — Extensión Antigravity (alternativa a Continue)

```bash
code --install-extension Antigravity.antigravity
```

En la configuración de VSCode (`settings.json`):

```json
{
  "antigravity.ollamaEndpoint": "http://localhost:11434",
  "antigravity.defaultModel": "qwen2.5-coder:7b",
  "antigravity.contextLength": 32768
}
```

### 5.3 OpenJarvis — Contenedor de Automatización OS

OpenJarvis es el componente de automatización a nivel de sistema operativo. Ya está incluido en el `docker-compose.yml` anterior.

```bash
# Verificar que el contenedor está corriendo
docker logs openjarvis --tail 50

# Acceder a la UI de OpenJarvis
# http://localhost:8080

# Configurar conexión a Ollama desde la UI:
# Settings → AI Backend → Ollama URL: http://ollama:11434
# Model: llama3.2

# Test de automatización básica
curl -X POST http://localhost:8080/api/task \
  -H "Content-Type: application/json" \
  -d '{"instruction": "lista los archivos en /mnt/ia_data", "model": "llama3.2"}'
```

> ⚠️ OpenJarvis requiere acceso al socket Docker (`/var/run/docker.sock`). Revisar implicaciones de seguridad si el servidor está expuesto en red.

---

## 6. Troubleshooting

### Error: `ECONNREFUSED` en conexión IDE ↔ Ollama

**Diagnóstico:**

```bash
# Verificar que Ollama escucha en el puerto correcto
ss -tlnp | grep 11434
# Debe mostrar: 0.0.0.0:11434

# Verificar variable OLLAMA_HOST
docker exec ollama env | grep OLLAMA_HOST
# o en nativo:
systemctl show ollama.service | grep Environment
```

**Solución:**

```bash
# Docker: verificar que el puerto está publicado
docker inspect ollama | grep -A5 Ports

# Nativo: el servicio debe reiniciarse después de cambiar override.conf
sudo systemctl daemon-reload && sudo systemctl restart ollama

# Firewall: permitir puerto si hay ufw activo
sudo ufw allow 11434/tcp
sudo ufw status
```

### Error: Permisos de escritura en `/mnt/ia_data`

```bash
# Identificar propietario actual
ls -la /mnt/ia_data

# Si el contenedor escribe como root (uid 0) y el bind mount es de otro usuario:
sudo chown -R 1000:1000 /mnt/ia_data
# uid 1000 es el primer usuario no-root en la mayoría de sistemas Ubuntu

# Verificar desde dentro del contenedor
docker exec -it ollama ls -la /root/.ollama
docker exec -it ollama touch /root/.ollama/test_write && echo "OK: escritura funciona"
```

### Error: VRAM insuficiente / modelo no carga

```bash
# Ver uso actual de VRAM
nvidia-smi --query-gpu=memory.used,memory.free,memory.total --format=csv

# Listar modelos cargados en memoria por Ollama
curl http://localhost:11434/api/ps | python3 -m json.tool

# Descargar modelo de VRAM sin eliminar del disco
curl -X DELETE http://localhost:11434/api/delete \
  -H "Content-Type: application/json" \
  -d '{"name":"llama3.2"}'
# Nota: esto elimina del disco. Para solo descargar de memoria, usar OLLAMA_KEEP_ALIVE=0

# Forzar uso de CPU para un modelo específico (override temporal)
docker exec -it ollama env CUDA_VISIBLE_DEVICES="" ollama run llama3.2

# Cambiar a modelo más pequeño si la VRAM no alcanza
ollama pull llama3.2:1b          # ~1.3GB
ollama pull qwen2.5-coder:3b     # ~2.0GB
```

### Error: Contenedor OpenClaw no conecta a Ollama

```bash
# Desde dentro del contenedor OpenClaw, probar conectividad
docker exec -it openclaw curl http://ollama:11434/api/tags

# Si falla: verificar que ambos contenedores están en la misma red
docker network inspect ia-local_ia_net | grep -A3 Containers

# Forzar recarga de la red
docker compose down && docker compose up -d
```

### Verificación completa del stack

```bash
# Script de health check rápido
echo "=== Ollama ===" && curl -s http://localhost:11434/api/tags | python3 -c "import sys,json; d=json.load(sys.stdin); print(f'Modelos: {len(d[\"models\"])}')"
echo "=== OpenClaw ===" && curl -s -o /dev/null -w "HTTP %{http_code}" http://localhost:3000
echo "=== OpenJarvis ===" && curl -s -o /dev/null -w "HTTP %{http_code}" http://localhost:8080
echo "=== VRAM ===" && nvidia-smi --query-gpu=name,memory.used,memory.total --format=csv,noheader 2>/dev/null || echo "Sin GPU NVIDIA"
echo "=== Disco IA ===" && df -h /mnt/ia_data
```

---

## Resumen de Puertos y Servicios

| Servicio | Puerto | Protocolo | Acceso |
|---|---|---|---|
| Ollama API | 11434 | HTTP/REST | IDE, OpenClaw, OpenJarvis |
| OpenClaw UI | 3000 | HTTP | Navegador |
| OpenJarvis UI | 8080 | HTTP | Navegador |

## Estructura Final en Disco

```
/mnt/ia_data/
├── ollama_models/      # Pesos de modelos LLM (~GB por modelo)
├── openclaw_data/      # Base de datos, conversaciones, usuarios
└── logs/               # Logs de OpenJarvis y servicios
```

---

*Stack validado sobre Ubuntu 22.04 LTS / 24.04 LTS. Docker Engine 25+. NVIDIA Driver 535+.*
