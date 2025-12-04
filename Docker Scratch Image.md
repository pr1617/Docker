# Docker Scratch Image – Complete Guide & Production-Ready Example

[![Docker](https://img.shields.io/badge/Docker-scratch-2496ED?style=for-the-badge&logo=docker&logoColor=white)](https://hub.docker.com/_/scratch)
[![Size](https://img.shields.io/badge/Image%20Size-%3C%2010MB-success?style=for-the-badge)]()
[![Security](https://img.shields.io/badge/Attack%20Surface-Minimal-brightgreen?style=for-the-badge)]()

The ultimate minimal Docker image: **zero OS, zero shell, zero bloat**.  
Perfect for statically compiled Go, Rust, Zig, or C++ apps.

---

## Why Use `scratch`?

| Benefit                 | Result                                            |
|-------------------------|---------------------------------------------------|
| Tiny images             | 5–15 MB (vs 50–800 MB with Alpine/Ubuntu)        |
| Lightning-fast startup  | Ideal for serverless, Kubernetes, CI/CD           |
| Minimal attack surface  | No shell, no package manager, no unnecessary bins|
| Lower costs             | Cheaper registry storage & bandwidth              |
| Full control            | You decide every single byte                      |

---

## Complete Production-Ready Example (Go)

### Project Structure
```
myapp/
├── main.go
├── go.mod
└── Dockerfile
```

### `main.go`
```go
package main

import (
    "fmt"
    "log"
    "net/http"
    "os"
    "time"
)

func handler(w http.ResponseWriter, r *http.Request) {
    hostname, _ := os.Hostname()
    fmt.Fprintf(w, "Hello from scratch!\nHost: %s\nTime: %s\n", hostname, time.Now().Format(time.RFC3339))
}

func main() {
    http.HandleFunc("/", handler)
    log.Println("Server starting on :8080")
    log.Fatal(http.ListenAndServe(":8080", nil))
}
```

### `go.mod`
```go
module myapp

go 1.23
```

### `Dockerfile` – The Complete Production Version
```dockerfile
# ---------- Builder Stage ----------
FROM golang:1.23-alpine AS builder

WORKDIR /app

# Install tools we need for certs, timezone & user creation
RUN apk add --no-cache git ca-certificates tzdata

# Create non-root user (UID/GID 1001)
RUN adduser -D -g '' -u 1001 appuser

# Copy source
COPY go.mod go.sum ./
RUN go mod download
COPY . .

# Build static binary (critical for scratch)
RUN CGO_ENABLED=0 GOOS=linux GOARCH=amd64 \
    go build -ldflags="-s -w -extldflags '-static'" -o myapp

# ---------- Final Scratch Image ----------
FROM scratch

# Copy only the absolute minimum required files
COPY --from=builder /etc/passwd           /etc/passwd
COPY --from=builder /etc/group            /etc/group
COPY --from=builder /etc/ssl/certs/ca-certificates.crt /etc/ssl/certs/ca-certificates.crt
COPY --from=builder /usr/share/zoneinfo   /usr/share/zoneinfo/
COPY --from=builder /tmp                  /tmp

# Copy the static binary
COPY --from=builder /app/myapp            /myapp

# Environment
ENV TZ=UTC

# Use non-root user
USER 1001:1001

# Expose port
EXPOSE 8080

# Run
ENTRYPOINT ["/myapp"]
```

### Build & Run
```bash
# Build (~6–10 MB final image)
docker build -t myapp-scratch:latest .

# Run
docker run -p 8080:8080 myapp-scratch:latest
```

Visit: http://localhost:8080

---

## What You Must Manually Add to `scratch`

| File/Path                             | Why You Need It                         | Copy Command Example                                   |
|---------------------------------------|-----------------------------------------|--------------------------------------------------------|
| `/etc/ssl/certs/ca-certificates.crt`  | HTTPS/TLS requests                      | `COPY --from=alpine:latest /etc/ssl/certs/...`         |
| `/usr/share/zoneinfo`                 | Correct time in logs/scheduling         | `COPY --from=builder /usr/share/zoneinfo ...`          |
| `/etc/passwd` + `/etc/group`          | Run as non-root                         | `COPY --from=builder /etc/passwd /etc/passwd`          |
| `/tmp`                                | Temporary files, some libs expect it    | `COPY --from=builder /tmp /tmp`                        |
| `/etc/nsswitch.conf`, `/etc/hosts`    | DNS resolution in restricted networks   | (optional)                                             |

---

## Real-World Projects Using `scratch`

- Prometheus
- K3s dashboard
- Many Cloud Native Buildpacks outputs
- Distroless alternatives (even smaller!)

---

## Quick Checklist Before Shipping

- [ ] Binary is **fully static** (`CGO_ENABLED=0`, `-extldflags '-static'`)
- [ ] Runs as non-root user
- [ ] CA certificates included (if app makes HTTPS calls)
- [ ] Timezone data included
- [ ] Tested in Kubernetes/OpenShift (some policies block root)
- [ ] Scanned with Trivy/Clair → usually **0 CVEs**

---
