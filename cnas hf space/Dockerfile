# Hugging Face Space build — one container: FastAPI serves the API and the
# built React UI on a single port. Differences from the repo's Dockerfile:
# Spaces run the container as uid 1000, so everything the app writes to
# (data/, output/, audit.jsonl) and the Whisper model cache must belong to
# that user. Neo4j stays optional: unset NEO4J_* runs file-mode.

# ── Stage 1: build the React UI ────────────────────────────────────────────
FROM node:20-slim AS ui
WORKDIR /build/ui
COPY ui/package.json ui/package-lock.json ./
RUN npm ci --no-audit --no-fund
COPY ui/ ./
RUN npm run build

# ── Stage 2: Python API + baked UI, running as uid 1000 ────────────────────
FROM python:3.14-slim
ENV PYTHONDONTWRITEBYTECODE=1 PYTHONUNBUFFERED=1
RUN useradd -m -u 1000 user
COPY requirements.txt /tmp/requirements.txt
RUN pip install --no-cache-dir -r /tmp/requirements.txt

USER user
ENV HOME=/home/user PATH=/home/user/.local/bin:$PATH
WORKDIR /home/user/app

# Pre-download Whisper as the runtime user, so the cache is readable at run
# time and the first dictation is not a ~500MB fetch. A smaller Space would
# use --build-arg WHISPER_MODEL_SIZE=base and the matching env var below.
ARG WHISPER_MODEL_SIZE=small
ENV WHISPER_MODEL_SIZE=${WHISPER_MODEL_SIZE} WHISPER_DEVICE=cpu WHISPER_COMPUTE_TYPE=int8
RUN python -c "from faster_whisper import WhisperModel; WhisperModel('${WHISPER_MODEL_SIZE}', device='cpu', compute_type='int8')"

COPY --chown=user . .
COPY --from=ui --chown=user /build/ui/dist ./ui/dist

EXPOSE 8000
CMD ["sh", "-c", "uvicorn backend.api.main:app --host 0.0.0.0 --port ${PORT:-8000}"]
