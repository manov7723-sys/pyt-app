# syntax=docker/dockerfile:1
# EKS nodes are amd64 — pin the platform; a silently-built arm64 image CrashLoopBackOffs.
FROM --platform=linux/amd64 python:3.13-slim AS builder

ENV PIP_DISABLE_PIP_VERSION_CHECK=1 \
    PIP_NO_CACHE_DIR=1

WORKDIR /app

# Install pinned deps into an isolated venv so the runtime stage copies one tree.
COPY requirements.txt .
RUN python -m venv /opt/venv \
    && /opt/venv/bin/pip install --no-cache-dir -r requirements.txt


FROM --platform=linux/amd64 python:3.13-slim

# Pod runs with readOnlyRootFilesystem; only /tmp is writable, so point
# HOME/TMPDIR there and stop Python from writing .pyc files at import time.
ENV PATH="/opt/venv/bin:$PATH" \
    PYTHONUNBUFFERED=1 \
    PYTHONDONTWRITEBYTECODE=1 \
    HOME=/tmp \
    TMPDIR=/tmp

WORKDIR /app

COPY --from=builder /opt/venv /opt/venv
COPY . .

# Bytecode must exist before the filesystem goes read-only.
RUN python -m compileall -q /app /opt/venv

# Numeric uid: Kubernetes runAsNonRoot rejects images whose USER is a name.
USER 1000

# uvicorn listens on 8000 (config default PORT=8000).
EXPOSE 8000

# Base image is slim (no curl/wget) — probe with the stdlib instead.
HEALTHCHECK --interval=30s --timeout=5s --start-period=10s --retries=3 \
    CMD python -c "import urllib.request; urllib.request.urlopen('http://127.0.0.1:8000/openapi.json', timeout=4)" || exit 1

# DB_URL and DB_NAME are runtime secrets — never baked; supply via pod spec.
CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]
