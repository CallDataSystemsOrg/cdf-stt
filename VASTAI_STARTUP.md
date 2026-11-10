# Vast.ai Instance Startup Guide

This guide explains how to quickly start the STT service on a Vast.ai instance using the automated startup script.

## Prerequisites

- Vast.ai instance running with the cdf-stt Docker image
- HuggingFace token (get from https://huggingface.co/settings/tokens)
- Accept pyannote model terms:
  - https://huggingface.co/pyannote/speaker-diarization-3.1
  - https://huggingface.co/pyannote/segmentation-3.0

## Quick Start

**HF_TOKEN is not in the Docker image.** The image is public, so the token is
supplied at runtime from a secret: set `HF_TOKEN` in the Vast.ai instance's
environment variables (or `docker run -e HF_TOKEN=...`) when creating it.
Without it the service runs in transcription-only mode.

Then SSH in and run the script:

```bash
# SSH into your Vast.ai instance
ssh -i ~/.ssh/vastai -p <PORT> root@<HOST>

# Run the startup script (HF_TOKEN comes from the instance environment)
cd /app
./start_stt_service.sh
```

### Alternative: One-liner from local machine

```bash
# Run everything in one command (replace PORT and HOST)
ssh -i ~/.ssh/vastai -p <PORT> root@<HOST> "cd /app && ./start_stt_service.sh"
```

### Manual Override (if needed)

If the instance was created without `HF_TOKEN`, export it in the shell from
your secret store before starting (never pass it as a command-line argument):

```bash
cd /app
read -rs HF_TOKEN && export HF_TOKEN
./start_stt_service.sh
```

## What the Script Does

The `start_stt_service.sh` script automatically:

1. **Stops existing processes** - Kills any running Redis, API, or Worker processes
2. **Starts Redis server** - Launches Redis on port 6379
3. **Starts API server** - Launches FastAPI on port 8000 (mapped to external port)
4. **Starts Worker** - Launches background worker for async job processing
5. **Verifies services** - Checks that all services started successfully
6. **Tests endpoints** - Hits health and queue stats endpoints
7. **Displays information** - Shows external API URL and available endpoints
8. **Shows logs** - Displays recent API and Worker logs

## Services Started

After running the script, you'll have:

| Service | Internal Port | Log File |
|---------|--------------|----------|
| Redis | 6379 | (stdout) |
| API | 8000 | /tmp/api.log |
| Worker | - | /tmp/worker.log |

## External Access

The API will be accessible at: `http://<EXTERNAL_IP>:<EXTERNAL_PORT>`

Vast.ai maps internal port 8000 to an external port (usually displayed in the script output).

Example: `http://76.121.3.151:34360`

## Available API Endpoints

- `GET /health` - Health check
- `GET /languages` - Supported languages
- `POST /transcribe` - Synchronous transcription
- `POST /transcribe/async` - Async transcription (returns job_id)
- `GET /jobs/{job_id}` - Get job status and results
- `GET /jobs` - List all jobs
- `GET /queue/stats` - Queue statistics
- `GET /metrics` - Prometheus metrics

## Testing the Service

### Test Health Endpoint
```bash
curl http://<EXTERNAL_IP>:<PORT>/health
```

### Test Transcription
```bash
curl -X POST http://<EXTERNAL_IP>:<PORT>/transcribe/async \
  -F "file=@audio.wav" \
  -F "enable_diarization=true" \
  -F "min_speakers=2" \
  -F "max_speakers=2"
```

### Check Job Results
```bash
curl http://<EXTERNAL_IP>:<PORT>/jobs/{job_id}
```

## Viewing Logs

```bash
# Real-time API logs
tail -f /tmp/api.log

# Real-time Worker logs
tail -f /tmp/worker.log

# Redis monitoring
redis-cli monitor
```

## Troubleshooting

### Script fails to start Redis
```bash
# Check if Redis is already running
ps aux | grep redis

# Manually start Redis
redis-server --daemonize yes --bind 0.0.0.0 --protected-mode no
```

### Script fails to start API
```bash
# Check API logs
tail -50 /tmp/api.log

# Manually start API
export HF_TOKEN=hf_YourTokenHere
export LD_LIBRARY_PATH=/lib/x86_64-linux-gnu:$LD_LIBRARY_PATH
cd /app
uvicorn api.main:app --host 0.0.0.0 --port 8000
```

### Script fails to start Worker
```bash
# Check worker logs
tail -50 /tmp/worker.log

# Manually start worker
export HF_TOKEN=hf_YourTokenHere
export LD_LIBRARY_PATH=/lib/x86_64-linux-gnu:$LD_LIBRARY_PATH
export PYTHONPATH=/app:$PYTHONPATH
cd /app
python -m app.worker
```

## Restarting Services

To restart all services, simply run the script again:

```bash
cd /app
./start_stt_service.sh hf_YourTokenHere
```

The script will automatically stop existing processes before starting new ones.
