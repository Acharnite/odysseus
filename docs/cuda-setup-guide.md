# Odysseus CUDA Setup Guide
## Få Odysseus til at køre med NVIDIA GPU og llama.cpp CUDA-support

### 1. Forudsætninger
- NVIDIA GPU med driver installeret
- Docker + docker compose
- NVIDIA Container Toolkit (`nvidia-container-toolkit`)

Tjek med:
```bash
nvidia-smi
docker info | grep -i nvidia
nvidia-ctk --version
```

### 2. Klon Odysseus
```bash
git clone https://github.com/…/odysseus.git
cd odysseus
```

### 3. Konfigurer .env
Kopér `.env.example` til `.env` og sæt:
```bash
COMPOSE_FILE=docker-compose.yml:docker/gpu.nvidia.yml
APP_BIND=0.0.0.0
APP_PORT=7200
```

### 4. Opdater Dockerfile
Erstat `FROM python:3.12-slim` med:

```dockerfile
FROM nvidia/cuda:12.4.1-devel-ubuntu22.04
```

Tilføj Python til apt-get:
```
    python3 \
    python3-pip \
    python3-venv \
```

Ret `pip install` til `pip3 install`.

### 5. Opdater docker/entrypoint.sh
Find blokken der sætter CUDA_HOME (linje 64-72 i original). Erstat den med:

```bash
if [ -x /usr/local/cuda/bin/nvcc ]; then
    export CUDA_HOME=/usr/local/cuda
else
    for cu in \
        /app/.local/lib/python*/site-packages/nvidia/cu13 \
        /app/.local/lib/python*/site-packages/nvidia/cu12 \
        /app/.local/lib/python*/site-packages/nvidia/cuda_nvcc; do
        if [ -x "$cu/bin/nvcc" ]; then
            export CUDA_HOME="$cu"
            break
        fi
    done
fi
```

### 6. Byg og start
```bash
docker compose build --no-cache odysseus
docker compose up -d
```

### 7. Verificer CUDA i containeren
```bash
docker compose exec odysseus nvcc --version
docker compose exec odysseus nvidia-smi
```

### 8. Installer llama-cpp-python med CUDA
Gå til Odysseus → Cookbook → Dependencies → Installer llama-cpp-python[server].  
Den skulle automatisk detektere CUDA og bygge med `-DGGML_CUDA=ON`.

Du skulle se:
```
[odysseus] CUDA nvcc + cudart found — building llama-server with CUDA (GPU) support...
```

### 9. Serve en model
Gå til Odysseus → Cookbook → Serve → Vælg en GGUF-model og kør.  
Brug `-ngl 99` for at offloade alle layers til GPU.

### 10. Fejlfinding

| Problem | Løsning |
|---------|---------|
| `nvcc not found` | Dockerfile bruger ikke CUDA-base image |
| `CUDA runtime library missing` | Entrypoint overskriver CUDA_HOME — tjek step 5 |
| `CUDA_HOME=/app/.local/...` | Entrypoint prioriterer pip-stub — tjek step 5 |
| Containeren starter ikke | Kør `docker compose logs odysseus` for fejl |
