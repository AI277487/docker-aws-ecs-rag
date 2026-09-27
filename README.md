# RhinoBot — Docker + AWS Deployment

Containerization and cloud deployment of **RhinoBot**, a citation-grounded medical
RAG service (FastAPI + ChromaDB, hybrid BM25 + vector retrieval). This repo
documents packaging the app as a Docker image and running it on **AWS ECS
(Fargate)** behind an Application Load Balancer.

Live demo: https://rhino.artai.in/ · Main project: https://github.com/AI277487

---

## What this shows

- Packaging a Python/FastAPI RAG service (heavy deps: PyTorch CPU,
  sentence-transformers, ChromaDB) into a reproducible Docker image.
- Pushing the image to **Amazon ECR** and running it on **ECS Fargate** behind an
  **Application Load Balancer**, with secrets injected at runtime.
- Cost-aware operation: capture proof, then tear down; the image stays in ECR for
  a ~10-minute redeploy.

## Pipeline

```
Local image  ->  Amazon ECR (store)  ->  ECS Fargate (run)  ->  ALB (public URL)
```

ECR stores the image. ECS/Fargate runs it as a container. The load balancer gives
it a public HTTPS address and routes traffic to it.

---

## Files

| File | Purpose |
|---|---|
| `Dockerfile` | Recipe to build the image (Python 3.14 slim, CPU PyTorch, uvicorn). |
| `.dockerignore` | Keeps secrets (`.env`), the local `venv/`, and caches out of the image. |
| `requirements.txt` | Exact pinned dependencies (generated with `pip freeze`). |

### Dockerfile notes
- **Base matches the app's Python** (`python:3.14-slim`) to avoid "works on the
  server, breaks in the container" bugs.
- **Layer caching:** `requirements.txt` is copied and installed *before* the app
  code, so code changes don't trigger a full dependency reinstall.
- **CPU PyTorch:** installed via `--extra-index-url https://download.pytorch.org/whl/cpu`
  (the `+cpu` build isn't on the default PyPI index).
- **`CMD` binds `0.0.0.0`**, not `127.0.0.1`, so the app is reachable from outside
  the container.

---

## Build & run locally

```bash
docker build -t rhino-bot .
docker run -d -p 8001:8000 --env-file .env rhino-bot
curl http://localhost:8001/
```

`-p 8001:8000` maps host port 8001 to the container's 8000 (host:container).
Secrets come in at runtime via `--env-file` — never baked into the image.

## Push to Amazon ECR

```bash
REGION=$(aws configure get region)
ACCOUNT=$(aws sts get-caller-identity --query Account --output text)

aws ecr create-repository --repository-name rhino-bot --region $REGION
aws ecr get-login-password --region $REGION | \
  docker login --username AWS --password-stdin $ACCOUNT.dkr.ecr.$REGION.amazonaws.com
docker tag rhino-bot:latest $ACCOUNT.dkr.ecr.$REGION.amazonaws.com/rhino-bot:latest
docker push $ACCOUNT.dkr.ecr.$REGION.amazonaws.com/rhino-bot:latest
```

## Deploy on ECS (Fargate)

Via **ECS Express Mode** (or a standard ECS service):
- Source: the ECR image `rhino-bot:latest`.
- Port: **8000** (must match the container's port, or the ALB health check fails).
- CPU / memory: **2 vCPU / 4 GB** (the model + retrieval index need the headroom).
- Environment variables: the app's secrets (API keys, etc.).
- ECS auto-provisions the load balancer, target group, listener, and task; the
  service exposes a public **Application URL**.

## Cost note

Fargate + an ALB bill hourly, 24/7. After validating and capturing screenshots,
delete the service to stop the cost — the image stays in ECR (near-free), so it
redeploys in ~10 minutes on demand.

---

## Stack

Docker · Amazon ECR · Amazon ECS (Fargate) · Application Load Balancer · IAM ·
FastAPI · Python
