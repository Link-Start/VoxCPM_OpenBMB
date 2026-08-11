# Docker Support for VoxCPM Training WebUI

Run the VoxCPM LoRA fine-tuning WebUI in a Docker container with full GPU support and nginx reverse proxy.

## Prerequisites

- Docker Engine 19.03+ with [NVIDIA Container Toolkit](https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/install-guide.html)
- NVIDIA GPU with CUDA 12.4+ compatible drivers
- At least 16 GB GPU VRAM (24 GB+ recommended for larger models)

## Quick Start

```bash
# From the project root directory:
docker compose -f docker/docker-compose.yml up --build
```

This starts:
- **training-webui** — the Gradio-based training interface on port 7860
- **nginx** — reverse proxy serving the WebUI at `http://localhost/webui/`

Access the WebUI at **http://localhost/webui/**.

## Volume Mounts

The compose file maps host directories to container paths. Create these directories at the project root before starting:

```
VoxCPM/
├── docker/
│   ├── docker-compose.yml
│   ├── Dockerfile
│   └── nginx.conf
├── models/              ← Pretrained model weights (or auto-downloaded via HF)
│   ├── openbmb__VoxCPM2/
│   └── openbmb__VoxCPM1.5/
├── data/                ← Training manifests + audio files
│   ├── train.jsonl
│   ├── val.jsonl        (optional)
│   └── audio/
│       ├── speaker1_001.wav
│       └── ...
├── lora/                ← LoRA training output (created automatically)
│   └── my-voice-2024/
│       ├── checkpoints/
│       ├── logs/
│       └── train_config.yaml
└── output/              ← Additional training artifacts
```

### Mount Reference

| Host Path | Container Path | Purpose |
|-----------|---------------|---------|
| `./models/` | `/app/models` | Pretrained model weights and HF cache (`HF_HOME`). Pre-populate with model dirs (e.g., `openbmb__VoxCPM2/`) or leave empty — models auto-download on first run and persist here. |
| `./data/` | `/app/data` | Training data. Put JSONL manifests and audio files here. In the WebUI, reference paths as `/app/data/train.jsonl`. |
| `./lora/` | `/app/lora` | LoRA checkpoint output. After training, find results in `lora/<run-name>/checkpoints/`. Also used to resume training from existing checkpoints. |
| `./output/` | `/app/output` | Miscellaneous training artifacts. |

### Training Data Format

The train manifest is a JSONL file where each line references an audio file:

```json
{"audio_path": "/app/data/audio/speaker1_001.wav", "text": "Hello world", "speaker": "speaker1"}
```

Use absolute container paths (`/app/data/...`) in your manifest so the container can find the files.

### Models

If `models/openbmb__VoxCPM2/` exists on the host, the app loads directly from that path — no network access needed. If the directory is empty or missing, `from_pretrained` falls back to `snapshot_download` from HuggingFace Hub.

The Dockerfile sets `HF_HOME=/app/models` so any Hub downloads land in the same mounted volume (matching the pattern in `deploy/Dockerfile.voxcpm-unified`). This means models persist across container restarts regardless of whether they were pre-populated or auto-downloaded.

**Recommended:** Pre-populate to avoid first-run download delay:

```bash
huggingface-cli download openbmb/VoxCPM2 --local-dir ./models/openbmb__VoxCPM2
```

The Dockerfile creates empty `/app/models`, `/app/lora`, `/app/output` directories, but the volume mounts override them with your host directories.

## Health Check

The nginx proxy responds with `200 OK` on `GET /` for load balancer health checks (AWS ALB, etc.). This is separate from the WebUI at `/webui/`.

```bash
curl http://localhost/
# OK
```

## Direct Access (no proxy)

If you want to bypass nginx and access Gradio directly:

```bash
docker compose -f docker/docker-compose.yml up --build training-webui
```

Set `GRADIO_ROOT_PATH=` (empty) in the compose file when running without the proxy, then access at `http://localhost:7860`.

## Building Manually

```bash
# Build the image
docker build -f docker/Dockerfile -t voxcpm-training .

# Run with GPU access (no reverse proxy)
docker run --gpus all -p 7860:7860 \
    -v ./models:/app/models \
    -v ./data:/app/data \
    -v ./lora:/app/lora \
    -v ./output:/app/output \
    voxcpm-training
```

## Environment Variables

| Variable | Default | Description |
|----------|---------|-------------|
| `GRADIO_SERVER_PORT` | `7860` | Port for the WebUI server |
| `GRADIO_ROOT_PATH` | `""` | URL prefix when behind a reverse proxy (e.g., `/webui`) |
| `FORWARDED_ALLOW_IPS` | `*` | IPs allowed to set `X-Forwarded-Proto`. Set to `*` when behind a Docker reverse proxy so Gradio generates `https://` file URLs. |

## Reverse Proxy

The included `docker-compose.yml` ships with an nginx reverse proxy that serves the WebUI at `/webui/`. The `GRADIO_ROOT_PATH=/webui` env var ensures Gradio generates correct URLs for assets and WebSocket connections.

### Custom nginx config

Edit `docker/nginx.conf` to change the location prefix or add TLS.

### Traefik Example (labels)

```yaml
labels:
  - "traefik.http.routers.voxcpm.rule=PathPrefix(`/webui`)"
  - "traefik.http.services.voxcpm.loadbalancer.server.port=7860"
```

## Viewing Training Logs

Training subprocess output is streamed to stdout, visible via:

```bash
docker compose -f docker/docker-compose.yml logs -f training-webui
```

## Troubleshooting

- **"no NVIDIA GPU detected"**: Ensure the NVIDIA Container Toolkit is installed and `docker run --gpus all nvidia-smi` works.
- **OOM errors**: Reduce batch size in the WebUI or use a GPU with more VRAM.
- **WebUI not accessible**: Check that port 80 (nginx) or 7860 (direct) isn't blocked by a firewall.
- **WebSocket errors behind proxy**: Ensure your proxy forwards `Upgrade` and `Connection` headers (the included nginx.conf handles this).
- **Health check failing**: Ensure nginx is running — `curl http://localhost/` should return `OK`.
- **Mixed-content / audio not playing over HTTPS**: Gradio generates `http://` file URLs because uvicorn doesn't trust the `X-Forwarded-Proto` header from the Docker bridge network. The compose file sets `FORWARDED_ALLOW_IPS=*` to fix this. If you run without compose, pass `-e FORWARDED_ALLOW_IPS=*` to `docker run`.
