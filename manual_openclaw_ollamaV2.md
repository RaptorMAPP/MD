# Manual Técnico: OpenClaw + Ollama — Despliegue Local con Soberanía de Datos

> **Audiencia:** Ingenieros de sistemas / DevOps con conocimientos Linux intermedios-avanzados.  
> **Objetivo:** Eliminar dependencias SaaS externas mediante IA local autohospedada.  
> **Stack:** Ollama (motor de inferencia) + OpenClaw (frontend/orquestación) + OpenJarvis (automatización/voz).

---

## Índice

- [Sección 0 — Preparación de Almacenamiento](#sección-0--preparación-de-almacenamiento-disco-nuevo)
- [Sección 1 — Despliegue Docker (Recomendado)](#sección-1--despliegue-con-docker-recomendado)
- [Sección 2 — Despliegue Nativo Linux](#sección-2--despliegue-nativo-linux)
- [Sección 3 — Configuración y Conectividad](#sección-3--configuración-y-conectividad)
- [Troubleshooting](#troubleshooting)

---

## Sección 0 — Preparación de Almacenamiento (Disco Nuevo)

> Ejecutar **antes** de cualquier instalación de software. Los modelos LLM ocupan entre 4 GB y 70 GB por modelo; el almacenamiento dedicado evita saturar el disco del sistema.

### 0.1 Identificar el disco nuevo

```bash
lsblk -o NAME,SIZE,TYPE,MOUNTPOINT,FSTYPE
```

Salida típica:

```
NAME   SIZE TYPE MOUNTPOINT  FSTYPE
sda    500G disk
├─sda1 512M part /boot/efi   vfat
└─sda2 499G part /           ext4
sdb    2T   disk              ← disco nuevo sin particionar
```

Confirmar que `/dev/sdb` no tiene particiones ni punto de montaje antes de continuar.

---

### 0.2 Particionar con fdisk

```bash
sudo fdisk /dev/sdb
```

Secuencia interactiva dentro de `fdisk`:

| Comando | Acción |
|---------|--------|
| `g` | Crear tabla de particiones GPT (recomendado sobre MBR para discos > 2 TB) |
| `n` | Nueva partición |
| `1` | Número de partición |
| `↵` (Enter) | Primer sector — aceptar valor por defecto |
| `↵` (Enter) | Último sector — usar todo el disco |
| `w` | Escribir cambios y salir |

Verificar resultado:

```bash
lsblk /dev/sdb
# sdb    2T   disk
# └─sdb1 2T   part
```

---

### 0.3 Formatear la partición

```bash
sudo mkfs.ext4 -L ai_data /dev/sdb1
```

El flag `-L ai_data` asigna una etiqueta legible. La operación tarda 1–3 minutos en discos grandes.

---

### 0.4 Crear punto de montaje y montar temporalmente

```bash
sudo mkdir -p /mnt/ai_data
sudo mount /dev/sdb1 /mnt/ai_data
```

Verificar:

```bash
df -h /mnt/ai_data
# /dev/sdb1   2.0T   28K  1.9T   1% /mnt/ai_data
```

---

### 0.5 Montaje persistente vía /etc/fstab

Obtener el UUID del disco (más robusto que `/dev/sdbX` que puede cambiar con reordenamientos de bus):

```bash
sudo blkid /dev/sdb1
# /dev/sdb1: LABEL="ai_data" UUID="a1b2c3d4-..." TYPE="ext4"
```

Añadir al final de `/etc/fstab`:

```bash
sudo nano /etc/fstab
```

```
# Disco dedicado IA local
UUID=a1b2c3d4-...   /mnt/ai_data   ext4   defaults,noatime   0   2
```

> `noatime` desactiva la escritura de marcas de acceso — mejora rendimiento en lectura intensiva de modelos.

Probar configuración sin reiniciar:

```bash
sudo umount /mnt/ai_data
sudo mount -a
df -h /mnt/ai_data  # debe aparecer montado
```

---

### 0.6 Permisos de escritura para el usuario actual

```bash
sudo chown -R $USER:$USER /mnt/ai_data
sudo chmod 755 /mnt/ai_data
```

Crear estructura de directorios base:

```bash
mkdir -p /mnt/ai_data/{ollama_models,openclaw_data,openclaw_config,openjarvis,backups}
```

---

## Sección 1 — Despliegue con Docker (Recomendado)

> **Ventajas sobre instalación nativa:** aislamiento de dependencias, rollback trivial, networking declarativo, portabilidad entre hosts.

### 1.1 Prerequisitos del sistema

#### Docker Engine

```bash
# Eliminar versiones antiguas
sudo apt remove docker docker-engine docker.io containerd runc -y

# Dependencias
sudo apt update && sudo apt install -y \
  ca-certificates curl gnupg lsb-release

# Repositorio oficial Docker
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | \
  sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg

echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] \
  https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

# Agregar usuario al grupo docker (requiere logout/login)
sudo usermod -aG docker $USER
```

Verificar:

```bash
docker --version
docker compose version
```

---

#### NVIDIA Container Toolkit (solo si tienes GPU NVIDIA)

Primero instalar/verificar drivers NVIDIA:

```bash
nvidia-smi  # si falla, instalar drivers
```

Si `nvidia-smi` falla:

```bash
sudo apt install -y ubuntu-drivers-common
sudo ubuntu-drivers autoinstall
sudo reboot
```

Tras confirmar que `nvidia-smi` funciona, instalar el toolkit:

```bash
# Repositorio NVIDIA Container Toolkit
curl -fsSL https://nvidia.github.io/libnvidia-container/gpgkey | \
  sudo gpg --dearmor -o /usr/share/keyrings/nvidia-container-toolkit-keyring.gpg

curl -s -L https://nvidia.github.io/libnvidia-container/stable/deb/nvidia-container-toolkit.list | \
  sed 's#deb https://#deb [signed-by=/usr/share/keyrings/nvidia-container-toolkit-keyring.gpg] https://#g' | \
  sudo tee /etc/apt/sources.list.d/nvidia-container-toolkit.list

sudo apt update
sudo apt install -y nvidia-container-toolkit

# Configurar Docker para usar NVIDIA runtime
sudo nvidia-ctk runtime configure --runtime=docker
sudo systemctl restart docker
```

Verificar acceso GPU en contenedores:

```bash
docker run --rm --gpus all nvidia/cuda:12.3.1-base-ubuntu22.04 nvidia-smi
```

---

### 1.2 Estructura de directorios del proyecto

```bash
mkdir -p ~/ai-stack
cd ~/ai-stack
```

---

### 1.3 Archivo docker-compose.yml

```bash
nano ~/ai-stack/docker-compose.yml
```

```yaml
version: "3.9"

networks:
  ai_net:
    driver: bridge
    ipam:
      config:
        - subnet: 172.30.0.0/24

volumes:
  ollama_models:
    driver: local
    driver_opts:
      type: none
      o: bind
      device: /mnt/ai_data/ollama_models

  openclaw_data:
    driver: local
    driver_opts:
      type: none
      o: bind
      device: /mnt/ai_data/openclaw_data

  openclaw_config:
    driver: local
    driver_opts:
      type: none
      o: bind
      device: /mnt/ai_data/openclaw_config

services:
  # ─────────────────────────────────────────────
  # Motor de inferencia: Ollama
  # ─────────────────────────────────────────────
  ollama:
    image: ollama/ollama:latest
    container_name: ollama
    restart: unless-stopped
    networks:
      ai_net:
        ipv4_address: 172.30.0.10
    ports:
      - "11434:11434"
    volumes:
      - ollama_models:/root/.ollama
    environment:
      - OLLAMA_HOST=0.0.0.0
      - OLLAMA_KEEP_ALIVE=24h
    # Habilitar GPU: descomentar si tienes NVIDIA
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              count: all
              capabilities: [gpu]
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:11434/api/tags"]
      interval: 30s
      timeout: 10s
      retries: 5

  # ─────────────────────────────────────────────
  # Frontend/Orquestador: OpenClaw
  # ─────────────────────────────────────────────
  openclaw:
    image: openclaw/openclaw:latest
    container_name: openclaw
    restart: unless-stopped
    depends_on:
      ollama:
        condition: service_healthy
    networks:
      ai_net:
        ipv4_address: 172.30.0.11
    ports:
      - "3000:3000"
    volumes:
      - openclaw_data:/app/data
      - openclaw_config:/app/config
    environment:
      - OLLAMA_BASE_URL=http://ollama:11434
      - WEBUI_SECRET_KEY=cambia_esta_clave_aleatoria_segura
      - DEFAULT_MODELS=llama3.2,qwen2.5-coder

  # ─────────────────────────────────────────────
  # Automatización/Voz: OpenJarvis
  # ─────────────────────────────────────────────
  openjarvis:
    image: openjarvis/openjarvis:latest
    container_name: openjarvis
    restart: unless-stopped
    depends_on:
      - ollama
      - openclaw
    networks:
      ai_net:
        ipv4_address: 172.30.0.12
    ports:
      - "8080:8080"
    volumes:
      - /mnt/ai_data/openjarvis:/data
    environment:
      - OLLAMA_HOST=http://ollama:11434
      - OPENCLAW_HOST=http://openclaw:3000
```

> **Nota GPU:** El bloque `deploy.resources.reservations` aplica con `docker compose` v2+. En modo legacy sustituir por `runtime: nvidia` al nivel del servicio.

---

### 1.4 Iniciar el stack

```bash
cd ~/ai-stack

# Primera vez — descarga imágenes
docker compose up -d

# Logs en tiempo real
docker compose logs -f

# Estado de servicios
docker compose ps
```

---

### 1.5 Descargar modelos LLM

```bash
docker exec ollama ollama pull llama3.2          # 2GB — propósito general
docker exec ollama ollama pull qwen2.5-coder     # 4GB — código
docker exec ollama ollama pull nomic-embed-text  # 274MB — embeddings/RAG

# Verificar
docker exec ollama ollama list
```

---

## Sección 2 — Despliegue Nativo Linux

> Elegir cuando se requiere máxima performance sin overhead de contenedores, o integración directa con systemd.

### 2.1 Drivers NVIDIA

```bash
nvidia-smi  # verificar driver activo

# Si no está instalado
sudo apt install -y ubuntu-drivers-common
sudo ubuntu-drivers autoinstall && sudo reboot

# Verificar carga del módulo
lsmod | grep nvidia
```

---

### 2.2 Instalación nativa de Ollama

```bash
curl -fsSL https://ollama.com/install.sh | sh
```

El script instala el binario en `/usr/local/bin/ollama` y crea el servicio systemd automáticamente.

**Redirigir modelos al disco dedicado:**

```bash
sudo systemctl stop ollama
sudo mkdir -p /mnt/ai_data/ollama_models
sudo chown ollama:ollama /mnt/ai_data/ollama_models

sudo nano /etc/systemd/system/ollama.service
```

Añadir en `[Service]`:

```ini
[Service]
Environment="OLLAMA_MODELS=/mnt/ai_data/ollama_models"
Environment="OLLAMA_HOST=0.0.0.0:11434"
Environment="OLLAMA_KEEP_ALIVE=24h"
Environment="LD_LIBRARY_PATH=/usr/local/cuda/lib64:/usr/lib/x86_64-linux-gnu"
```

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now ollama
systemctl status ollama
curl http://localhost:11434/api/tags
```

---

### 2.3 Instalación nativa de OpenClaw

```bash
sudo apt update && sudo apt install -y \
  git python3 python3-pip python3-venv build-essential

sudo git clone https://github.com/openclaw/openclaw.git /opt/openclaw
sudo chown -R $USER:$USER /opt/openclaw
cd /opt/openclaw

python3 -m venv venv
source venv/bin/activate
pip install --upgrade pip
pip install -r requirements.txt

cp .env.example .env
nano .env
```

Variables mínimas en `.env`:

```dotenv
OLLAMA_BASE_URL=http://127.0.0.1:11434
DATA_DIR=/mnt/ai_data/openclaw_data
CONFIG_DIR=/mnt/ai_data/openclaw_config
PORT=3000
WEBUI_SECRET_KEY=genera_clave_con_python3_c_secrets_token_hex_32
```

---

### 2.4 Servicios systemd

#### `/etc/systemd/system/openclaw.service`

```ini
[Unit]
Description=OpenClaw AI Frontend
After=network.target ollama.service
Requires=ollama.service

[Service]
Type=simple
User=TUUSUARIO
WorkingDirectory=/opt/openclaw
ExecStart=/opt/openclaw/venv/bin/python main.py
Restart=on-failure
RestartSec=5
StandardOutput=journal
StandardError=journal
Environment="PATH=/opt/openclaw/venv/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin"

[Install]
WantedBy=multi-user.target
```

#### `/etc/systemd/system/openjarvis.service`

```ini
[Unit]
Description=OpenJarvis Automation Engine
After=network.target ollama.service openclaw.service

[Service]
Type=simple
User=TUUSUARIO
WorkingDirectory=/opt/openjarvis
ExecStart=/opt/openjarvis/venv/bin/python jarvis.py
Restart=on-failure
RestartSec=5
Environment="OLLAMA_HOST=http://127.0.0.1:11434"
Environment="OPENCLAW_HOST=http://127.0.0.1:3000"

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now openclaw openjarvis
systemctl status openclaw openjarvis
```

---

## Sección 3 — Configuración y Conectividad

### 3.1 Onboarding y selección de modelos

Abrir `http://localhost:3000`. El wizard solicita:

1. Crear cuenta de administrador local.
2. Conectar a Ollama: `http://127.0.0.1:11434` (nativo) o `http://ollama:11434` (Docker interno).
3. Seleccionar modelos por defecto.

| Modelo | Caso de uso | VRAM mínima |
|--------|------------|-------------|
| `llama3.2` | Chat general | 4 GB |
| `llama3.2:1b` | Hardware limitado | 1 GB |
| `qwen2.5-coder` | Generación de código | 6 GB |
| `qwen2.5-coder:1.5b` | Autocompletado ligero | 2 GB |
| `nomic-embed-text` | RAG / búsqueda semántica | <1 GB |

---

### 3.2 Integración VS Code — Continue.dev

```bash
code --install-extension Continue.continue
```

Configuración `~/.continue/config.json`:

```json
{
  "models": [
    {
      "title": "Llama 3.2 Local",
      "provider": "ollama",
      "model": "llama3.2",
      "apiBase": "http://localhost:11434"
    },
    {
      "title": "Qwen2.5 Coder Local",
      "provider": "ollama",
      "model": "qwen2.5-coder",
      "apiBase": "http://localhost:11434"
    }
  ],
  "tabAutocompleteModel": {
    "title": "Qwen Coder Autocomplete",
    "provider": "ollama",
    "model": "qwen2.5-coder:1.5b",
    "apiBase": "http://localhost:11434"
  },
  "embeddingsProvider": {
    "provider": "ollama",
    "model": "nomic-embed-text",
    "apiBase": "http://localhost:11434"
  }
}
```

| Atajo | Acción |
|-------|--------|
| `Ctrl+L` | Chat lateral |
| `Ctrl+I` | Editar selección |
| `Ctrl+Shift+R` | Refactorizar |

---

### 3.3 Integración VS Code — Antigravity

1. Instalar desde el marketplace de VS Code.
2. `Ctrl+,` → buscar `Antigravity`.
3. Configurar:
   - **Endpoint:** `http://localhost:11434/v1`
   - **Model:** nombre exacto del modelo (ej. `llama3.2`)
   - **API Type:** `OpenAI Compatible`

---

### 3.4 Acceso remoto desde la red local

```bash
# En Continue.dev / Antigravity — reemplazar localhost por IP del servidor
"apiBase": "http://192.168.1.50:11434"

# Abrir puertos necesarios
sudo ufw allow 11434/tcp   # Ollama API
sudo ufw allow 3000/tcp    # OpenClaw UI
sudo ufw allow 8080/tcp    # OpenJarvis
sudo ufw reload
```

---

### 3.5 OpenJarvis — Automatización y voz

**Webhook de automatización:**

```bash
curl -X POST http://localhost:8080/api/trigger \
  -H "Content-Type: application/json" \
  -d '{"action": "summarize", "input": "texto a procesar"}'
```

**Audio en Docker:**

```yaml
# En servicio openjarvis — docker-compose.yml
devices:
  - /dev/snd:/dev/snd
environment:
  - PULSE_SERVER=unix:/run/user/1000/pulse/native
volumes:
  - /run/user/1000/pulse:/run/user/1000/pulse
```

**Audio en instalación nativa:**

```bash
groups $USER | grep audio
# Si no está: sudo usermod -aG audio $USER && logout
```

---

## Troubleshooting

### T1 — Connection refused: 127.0.0.1 vs 0.0.0.0

**Síntoma:** OpenClaw falla al conectar con Ollama.

**Diagnóstico:**

```bash
ss -tlnp | grep 11434
# Si muestra 127.0.0.1:11434 → problema confirmado
```

**Solución nativa:**

```bash
sudo systemctl edit ollama.service
# Añadir:
[Service]
Environment="OLLAMA_HOST=0.0.0.0:11434"

sudo systemctl restart ollama
ss -tlnp | grep 11434  # debe mostrar 0.0.0.0:11434
```

**Solución Docker:** Verificar `OLLAMA_HOST=0.0.0.0` en el servicio `ollama` y que `openclaw` use `http://ollama:11434` (nombre del servicio, no `localhost`).

---

### T2 — GPU no asignada al contenedor Docker

**Diagnóstico:**

```bash
docker exec ollama nvidia-smi
# Si falla: GPU no visible en el contenedor
```

**Pasos de resolución:**

```bash
# 1. Verificar toolkit
nvidia-ctk --version

# 2. Verificar runtime en Docker
docker info | grep -i runtime
# debe incluir: Runtimes: nvidia runc

# 3. Reconfigurar si falta
sudo nvidia-ctk runtime configure --runtime=docker
sudo systemctl restart docker

# 4. Prueba directa
docker run --rm --gpus all nvidia/cuda:12.3.1-base-ubuntu22.04 nvidia-smi
```

---

### T3 — Permisos de escritura en /mnt/ai_data

**Diagnóstico:**

```bash
ls -la /mnt/ai_data
mount | grep ai_data  # verificar que está montado
```

**Solución:**

```bash
# Servicio ollama (usuario del sistema 'ollama')
sudo chown -R ollama:ollama /mnt/ai_data/ollama_models

# Otros servicios (tu usuario)
sudo chown -R $USER:$USER /mnt/ai_data/openclaw_data
sudo chown -R $USER:$USER /mnt/ai_data/openclaw_config

# Docker (contenedores corren como root internamente)
sudo chmod -R 755 /mnt/ai_data

# Verificar fstab
sudo findmnt --verify /mnt/ai_data
```

---

### T4 — Modelo cargado en CPU en lugar de GPU

**Diagnóstico:**

```bash
ollama ps
# NAME     SIZE    PROCESSOR  ← si dice "CPU" hay problema
```

**Solución:**

```bash
# Verificar LD_LIBRARY_PATH
echo $LD_LIBRARY_PATH

# Verificar en logs de Ollama
journalctl -u ollama -n 50 | grep -i "gpu\|nvidia\|cuda"
# Buscar: "Detected NVIDIA GPU"

# Si no aparece, añadir a la unidad systemd
sudo systemctl edit ollama.service
[Service]
Environment="LD_LIBRARY_PATH=/usr/local/cuda/lib64:/usr/lib/x86_64-linux-gnu"

sudo systemctl restart ollama
```

---

### T5 — Health check integral del stack

```bash
#!/bin/bash
echo "=== Ollama API ==="
curl -s http://localhost:11434/api/tags | python3 -m json.tool | head -20

echo "=== OpenClaw UI ==="
curl -s -o /dev/null -w "HTTP Status: %{http_code}\n" http://localhost:3000

echo "=== OpenJarvis API ==="
curl -s -o /dev/null -w "HTTP Status: %{http_code}\n" http://localhost:8080/health

echo "=== Disco /mnt/ai_data ==="
df -h /mnt/ai_data

echo "=== GPU ==="
nvidia-smi --query-gpu=name,memory.used,memory.free,utilization.gpu \
  --format=csv,noheader 2>/dev/null || echo "nvidia-smi no disponible"
```

---

*Manual técnico — automatiza-Ti | Stack IA Local v1.0*
