# Container Security 101 – Python  
**Now with real-world Python example**

Secure containers = minimal + hardened + verified + monitored

## Example: Production-Grade Secure Python Dockerfile (FastAPI / Flask / Django)

```dockerfile
# ================ BUILD STAGE ================
FROM python:3.12-slim-bookworm AS builder

# Install only build dependencies
RUN apt-get update && apt-get install -y --no-install-recommends \
    build-essential \
    gcc \
    && rm -rf /var/lib/apt/lists/*

# Create non-root user early (used for pip cache)
ARG UID=10001
RUN adduser \
    --disabled-password \
    --gecos "" \
    --home "/app" \
    --shell "/sbin/nologin" \
    --no-create-home \
    --uid "${UID}" \
    appuser

WORKDIR /app

# Install dependencies into /.venv (will be copied later)
COPY requirements.txt .
RUN pip install --no-cache-dir --upgrade pip setuptools wheel \
    && pip install --no-cache-dir --user -r requirements.txt

# ================ FINAL STAGE ================
# Chainguard Python = tiny (~45 MB), zero CVEs, non-root by default
FROM cgr.dev/chainguard/python:latest-dev AS runtime

# Copy only runtime files
WORKDIR /app
COPY --from=builder /etc/passwd /etc/passwd
COPY --from=builder /etc/group /etc/group

# Copy virtualenv from builder
COPY --from=builder --chown=65532:65532 /root/.local /home/nonroot/.local

# Copy source code (last = better layer caching)
COPY --chown=65532:65532 . .

# Use built-in nonroot user (UID 65532)
USER 65532

# Make .local bin available
ENV PATH="/home/nonroot/.local/bin:${PATH}"

EXPOSE 8000

# Proper signal handling + healthcheck
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
    CMD ["python", "-c", "import requests; requests.get('http://localhost:8000/health')"]

ENTRYPOINT ["uvicorn" if your app uses FastAPI else "gunicorn"]
CMD ["app.main:app", "--workers", "2", "--worker-class", "uvicorn.workers.UvicornWorker", "--bind", "0.0.0.0:8000"]
```

### Alternative: Even Smaller with Distroless (~25–35 MB)

```dockerfile
FROM python:3.12-slim AS builder
# ... same build steps above ...

FROM gcr.io/distroless/python3-debian12
COPY --from=builder /root/.local /home/nonroot/.local
COPY --from=builder /etc/passwd /etc/passwd
COPY . /app
WORKDIR /app
USER nonroot
ENV PATH="/home/nonroot/.local/bin:${PATH}"
EXPOSE 8000
ENTRYPOINT ["uvicorn"]
CMD ["app.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

### Alternative: Pure Scratch (smallest possible, ~15–25 MB compressed)

Only works if you build a fully static PyInstaller or Nuitka binary (advanced).

## Recommended requirements.txt snippet (secure defaults)

```txt
fastapi==0.115.*
uvicorn[standard]==0.30.*
gunicorn==23.*
# Pin digests in production via pip-tools or poetry
# Example:
fastapi==0.115.2 --hash=sha256:abc123...
```

## Kubernetes / Docker Runtime Hardening (add this!)

```yaml
securityContext:
  runAsNonRoot: true
  runAsUser: 65532
  runAsGroup: 65532
  capabilities:
    drop: ["ALL"]
  readOnlyRootFilesystem: true
  allowPrivilegeEscalation: false
  seccompProfile:
    type: RuntimeDefault
resources:
  limits:
    memory: "256Mi"
    cpu: "500m"
```

## Final Checklist for Your Python Containers

```markdown
[ ] python:slim or chainguard/python base
[ ] Multi-stage build
[ ] Non-root user (65532)
[ ] Dependencies installed as --user or copied from .local
[ ] No secrets in image/layers
[ ] requirements.txt pinned (ideally with hashes)
[ ] Image scanned (trivy scan image --scanners vuln .)
[ ] Image signed with cosign
[ ] Read-only root + resource limits
[ ] Healthcheck defined
```




