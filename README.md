# Wyoming Whisper — Intel GPU (Iris Xe / Arc)

Wyoming-compatible speech-to-text using [whisper.cpp](https://github.com/ggml-org/whisper.cpp) with Intel GPU acceleration via SYCL/Level Zero. Tested on Intel Iris Xe (i7-1360P, 96 EU).

**Latency:** ~3s for German voice commands (vs. ~7.5s on CPU with faster-whisper).

## Architecture

```
Home Assistant / App
        │ Wyoming TCP (port 10300)
        ▼
  wyoming-api          ← translates Wyoming protocol → HTTP
        │ HTTP POST /inference
        ▼
  whisper-cpp          ← whisper.cpp server with Intel SYCL backend
        │ /dev/dri (Level Zero)
        ▼
  Intel Iris Xe / Arc GPU
```

## Requirements

- Intel GPU with Level Zero support (Iris Xe 11th gen+, Arc)
- Docker with Compose v2
- `/dev/dri/renderD128` accessible

## Setup

### 1. Create external Docker network

```bash
docker network create wyoming-net
```

> If you already have an existing Docker network (e.g. a Swarm overlay `swarm-net`), create a separate bridge network named `wyoming-net` — attaching standalone Compose containers to Swarm overlay networks is unreliable.

### 2. Configure

```bash
cp .env.example .env
# Edit .env:
# - RENDER_GROUP_ID: run `getent group render | cut -d: -f3`
# - WHISPER_PROMPT: adapt to your smart home vocabulary
# - MODELS_DIR: where to store the GGML model file
```

### 3. Download the German model

```bash
mkdir -p ./models
wget https://huggingface.co/cstr/whisper-large-v3-turbo-german-ggml/resolve/main/ggml-model-q5_0.bin \
     -O ./models/ggml-large-v3-q5_0.bin
```

For other languages use any compatible GGML model from [ggerganov/whisper.cpp](https://huggingface.co/ggerganov/whisper.cpp).

### 4. Build and start

```bash
# Option A: build locally (~60 min for whisper-cpp due to oneAPI)
docker compose build
docker compose up -d whisper-cpp wyoming-api

# Option B: pull pre-built images (if available in ghcr.io)
docker compose pull whisper-cpp wyoming-api
docker compose up -d whisper-cpp wyoming-api
```

### 5. Point your app to the Wyoming endpoint

```
host: <server-ip>
port: 10300  # or WYOMING_PORT from .env
```

## Key configuration options (.env)

| Variable | Default | Description |
|---|---|---|
| `WHISPER_MODEL` | `large-v3-q5_0` | GGML model name (without `ggml-` / `.bin`) |
| `WHISPER_LANG` | `de` | Language code |
| `WHISPER_BEAM_SIZE` | `3` | Beam size (1=fastest, 3=recommended) |
| `WHISPER_AUDIO_CTX` | `768` | Encoder context frames (~7.5s); use 3000 for full 30s |
| `WHISPER_PROMPT` | *(see .env.example)* | Initial prompt for domain vocabulary |
| `RENDER_GROUP_ID` | `109` | Host render group id for GPU access |
| `WYOMING_PORT` | `10300` | Exposed Wyoming port |
| `MODELS_DIR` | `./models` | Path to model directory on host |

## What was patched

The upstream [wyoming-whisper-api-client](https://github.com/ser/wyoming-whisper-api-client) does not forward `initial_prompt` in the HTTP request to whisper-server. The patched `wyoming-api/handler.py` and `wyoming-api/__main__.py` add `--initial-prompt` CLI support and include it in every inference request.

## Credits

- [tannisroot/wyoming-whisper-cpp-intel-gpu-docker](https://github.com/tannisroot/wyoming-whisper-cpp-intel-gpu-docker) — original Intel GPU whisper.cpp setup
- [ser/wyoming-whisper-api-client](https://github.com/ser/wyoming-whisper-api-client) — Wyoming ↔ HTTP adapter
- [cstr/whisper-large-v3-turbo-german-ggml](https://huggingface.co/cstr/whisper-large-v3-turbo-german-ggml) — German GGML model
