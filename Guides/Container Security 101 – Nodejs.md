# Container Security 101 – Nodejs  
**Now with production-grade Node.js example**

Secure containers = minimal + hardened + verified + monitored

## Example: Ultra-Secure Node.js Dockerfile (Express / Next.js / NestJS)

```dockerfile
# ================ BUILD STAGE ================
FROM node:20-alpine AS builder

# Fail build on any npm audit critical issues (optional but recommended)
RUN npm config set audit-level high

WORKDIR /app

# Install only what's needed for building
COPY package*.json ./
RUN npm ci --only=production && npm cache clean --force

# Copy source after deps (better caching)
COPY . .

# Build if needed (Next.js, Vite, etc.)
# RUN npm run build

# ================ FINAL STAGE – Tiny & Secure ================
# Chainguard Node = ~45–55 MB, zero CVEs, non-root by default
FROM cgr.dev/chainguard/node:latest

WORKDIR /app

# Copy node_modules and built app from builder
COPY --from=builder --chown=65532:65532 /app/node_modules ./node_modules
COPY --from=builder --chown=65532:65532 /app .

# Explicit non-root user (Chainguard default = 65532)
USER 65532

# Remove potential attack vectors
RUN rm -rf /tmp/* /root/.npm || true

EXPOSE 3000

# Use non-root port if possible (>1024)
# Or use --cap-drop=NET_BIND_SERVICE if binding to 80/443

ENV NODE_ENV=production
ENV PORT=3000

HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
    CMD ["node", "healthcheck.js"] || wget --no-verbose --tries=1 --spider http://localhost:3000/health || exit 1

ENTRYPOINT ["node"]
CMD ["server.js"]        # or: "dist/main.js", ".output/server/index.mjs", "node_modules/.bin/next start", etc.
```

### Alternative A: Distroless Node (~30–40 MB)

```dockerfile
FROM node:20-alpine AS builder
# ... same as above ...

FROM gcr.io/distroless/nodejs20-debian12
COPY --from=builder /app /app
WORKDIR /app
USER nonroot
EXPOSE 3000
ENV NODE_ENV=production
ENTRYPOINT ["/nodejs/bin/node"]
CMD ["server.js"]
```

### Alternative B: Pure Alpine (still excellent, ~60–80 MB)

```dockerfile
FROM node:20-alpine AS final
WORKDIR /app
COPY --from=builder /app ./
USER node               # built-in alpine user
EXPOSE 3000
CMD ["node", "server.js"]
```

## Recommended package.json security settings

```json
{
  "scripts": {
    "preinstall": "npx only-allow pnpm",   // or npm/yarn
    "postinstall": "npm audit --audit-level=high || exit 1"
  },
  "engines": {
    "node": ">=20.12.0 <21"
  }
}
```

Use `npm ci` + lockfile v3 or `pnpm-lock.yaml` with hashes in production.

## Kubernetes / Docker Runtime Hardening

```yaml
securityContext:
  runAsNonRoot: true
  runAsUser: 65532
  runAsGroup: 65532
  capabilities:
    drop:
      - ALL
  readOnlyRootFilesystem: true
  allowPrivilegeEscalation: false
  seccompProfile:
    type: RuntimeDefault
resources:
  limits:
    memory: "256Mi"
    cpu: "500m"
```

## Final Node.js Security Checklist

```markdown
[ ] Base: chainguard/node or distroless/nodejs
[ ] Multi-stage build
[ ] npm ci --only=production
[ ] Non-root user (65532 or node)
[ ] NODE_ENV=production
[ ] No devDependencies in final image
[ ] package-lock.json / pnpm-lock.yaml with integrity hashes
[ ] Image scanned & no critical/high CVEs
[ ] Image signed with cosign
[ ] Read-only filesystem + resource limits
[ ] Healthcheck present
[ ] No .env files or secrets baked in
```

