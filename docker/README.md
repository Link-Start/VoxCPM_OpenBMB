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
    -v ./lora:/app/lora \
    -v ./output:/app/output \
    voxcpm-training
```

## Model Weights

Models are **auto-downloaded** from HuggingFace Hub on first use. The `/app/models` volume persists them across container restarts so they don't need to be re-downloaded.

To pre-populate (avoids download at startup):

```
models/
├── openbmb__VoxCPM2/       # VoxCPM2 (preferred)
└── openbmb__VoxCPM1.5/     # VoxCPM1.5 (fallback)
```

## Environment Variables

| Variable | Default | Description |
|----------|---------|-------------|
| `GRADIO_SERVER_PORT` | `7860` | Port for the WebUI server |
| `GRADIO_ROOT_PATH` | `""` | URL prefix when behind a reverse proxy (e.g., `/webui`) |

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

## Volumes

| Mount Point | Purpose |
|-------------|---------|
| `/app/models` | Pre-trained model weights (read-only OK) |
| `/app/lora` | LoRA checkpoints — training output is saved here |
| `/app/output` | Additional training artifacts |

## Troubleshooting

- **"no NVIDIA GPU detected"**: Ensure the NVIDIA Container Toolkit is installed and `docker run --gpus all nvidia-smi` works.
- **OOM errors**: Reduce batch size in the WebUI or use a GPU with more VRAM.
- **WebUI not accessible**: Check that port 80 (nginx) or 7860 (direct) isn't blocked by a firewall.
- **WebSocket errors behind proxy**: Ensure your proxy forwards `Upgrade` and `Connection` headers (the included nginx.conf handles this).
