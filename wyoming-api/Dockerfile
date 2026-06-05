FROM debian:bookworm-slim

SHELL ["/bin/bash", "-o", "pipefail", "-c"]

RUN \
    apt-get update \
    && apt-get install -y --no-install-recommends \
        git \
        netcat-traditional \
        python3-pip \
        python3-venv

RUN \
    git clone https://github.com/ser/wyoming-whisper-api-client.git wyoming-whisper-api-client \
    && cd wyoming-whisper-api-client \
    && script/setup

WORKDIR /wyoming-whisper-api-client

COPY handler.py /wyoming-whisper-api-client/wyoming_whisper_api_client/handler.py
COPY __main__.py /wyoming-whisper-api-client/wyoming_whisper_api_client/__main__.py
