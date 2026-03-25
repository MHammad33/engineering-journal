# Diagnosing "Failed to find Server Action" in Production Docker Deployments

**Date:** 2026-03-24
**Category:** Frontend / DevOps
**Status:** Completed
**Next.js:** 15.5.7

---

## 1. The Challenge

**Context:** A Next.js application deployed via Docker in production was intermittently throwing:
```
Error: Failed to find Server Action "x".
This request might be from an older or newer deployment.
```

The error could not be reproduced locally in development.

**The wall:** The error only surfaces under a specific combination of conditions: a user must have an old page open in the browser and a new Docker deployment must have happened in the background. Two reasons made it invisible in every non-production environment:

- `next dev` skips Server Action encryption entirely, this class of error cannot be triggered in dev mode.
- Local Docker builds reuse Next.js build cache (valid for 14 days), which keeps action IDs stable between builds and masks the issue.
- Production CI runs `docker build --no-cache`, which invalidates the Next.js build cache on every deployment, the exact condition that triggers the error.

---

## 2. Constraints

### Technical

- Could not test directly against the production project, had to reproduce in an isolated new project.
- Docker builds ran with `--no-cache`, invalidating Next.js build cache on every deployment.
- No `NEXT_SERVER_ACTIONS_ENCRYPTION_KEY` set, meaning both the encryption key and module IDs were non-deterministic on every build.

### Business

- Error was intermittent and user-facing, presenting as a silent UX failure with no visible client-side error boundary.
- Required reproducing a production-only condition in isolation, without access to production infrastructure.

---

## 3. Root Cause

The error has two independent sources of non-determinism. Both must be fixed. Either alone is not sufficient.

### Source 1: Missing NEXT_SERVER_ACTIONS_ENCRYPTION_KEY

Without this key, Next.js generates a random salt at build time. This salt is used to derive Server Action IDs and encrypt action payloads. Every `docker build --no-cache` produces a new salt and therefore new action IDs. The browser tab holds the old ID which no longer exists on the new server.

### Source 2: Non-deterministic webpack module IDs

Even with a stable encryption key, action IDs also incorporate the webpack module ID assigned to each file during bundling. Module IDs are assigned by webpack based on internal counters that vary between builds unless Next.js is given a stable build ID. Without it, consecutive builds produce different module IDs and therefore different action IDs:
```
// Two builds with the same code and same encryption key:
v1: moduleId: 2590,  actionId: 40a46a8d64faa8bc...
v2: moduleId: 4814,  actionId: 402b4586cbc43a64...
```

The Next.js `buildId` feeds into webpack's module ID generation. It is randomly generated on every build unless explicitly pinned via `generateBuildId` in `next.config.ts`.

> **Important:** `server-reference-manifest.js` stores the encryption key as the literal string `process.env.NEXT_SERVER_ACTIONS_ENCRYPTION_KEY`, not the actual key value. Next.js intentionally defers reading it to runtime via `process.env`. The key must be present at both build time (for stable ID generation) and runtime (for decryption).

---

## 4. Decision Matrix

| Option | Pros | Cons | Decision |
|---|---|---|---|
| Upgrade Next.js | May include internal fixes | Doesn't address root cause, risk of regressions | Rejected as sole fix |
| `NEXT_SERVER_ACTIONS_ENCRYPTION_KEY` + `generateBuildId` | Directly addresses both root causes. Single env var + one config line. No risk to existing functionality. | Key must be stored as a permanent CI/CD secret and never rotated unless prepared to invalidate active sessions. | ✅ Accepted |
| Force client refresh on deploy | Eliminates stale clients entirely | Requires additional infrastructure (deployment ID header check, service worker) | Considered as complementary |

---

## 5. Implementation

### Step 1: Generate a stable encryption key
```bash
openssl rand -base64 32

# Store the output in .env.production (never commit this file)
# NEXT_SERVER_ACTIONS_ENCRYPTION_KEY=<your-stable-base64-32-byte-key>
```

Next.js reads `.env.production` natively during `next build`, so the key is guaranteed to be present when action IDs are generated. It is also read at runtime via `process.env` for decryption.

### Step 2: Pin the build ID in next.config.ts
```ts
import type { NextConfig } from 'next';

const nextConfig: NextConfig = {
  generateBuildId: async () => process.env.GIT_HASH ?? 'fallback-id',
};

export default nextConfig;
```

### Step 3 — Update the Dockerfile

`ARG` and `ENV` for both variables must appear before `npm run build`:
```dockerfile
FROM node:22-alpine
WORKDIR /app
ENV NODE_ENV=production
COPY package.json package-lock.json ./
RUN npm ci --include=dev
COPY . .

ARG NEXT_SERVER_ACTIONS_ENCRYPTION_KEY
ARG GIT_HASH
ENV NEXT_SERVER_ACTIONS_ENCRYPTION_KEY=$NEXT_SERVER_ACTIONS_ENCRYPTION_KEY
ENV GIT_HASH=$GIT_HASH

RUN rm -rf /app/.next && npm run build

ENV HOSTNAME="0.0.0.0"
ENV PORT=3000
EXPOSE 3000
CMD ["npm", "start"]
```

### Step 4 — Build with stable args
```bash
docker build --no-cache \
  --build-arg "GIT_HASH=$(git rev-parse HEAD)" \
  -t myapp:latest .

# NEXT_SERVER_ACTIONS_ENCRYPTION_KEY is read from .env.production
# GIT_HASH pins webpack module IDs via generateBuildId
```

### Verification

Confirm action IDs are identical across consecutive builds:
```bash
docker run --rm myapp:v1 sh -c 'cat /app/.next/server/server-reference-manifest.js'
docker run --rm myapp:v2 sh -c 'cat /app/.next/server/server-reference-manifest.js'

# Both should show identical actionId and moduleId values
```

---

## 6. Key Learnings

- `next dev` skips Server Action encryption entirely — this class of error is impossible to reproduce locally in dev mode.
- Next.js build cache keeps action IDs stable between local builds, masking the issue. Production CI's `--no-cache` flag is what exposes it.
- `NEXT_SERVER_ACTIONS_ENCRYPTION_KEY` alone does not fix the issue. The action ID is also derived from webpack module IDs, which are non-deterministic unless `generateBuildId` is pinned. Both fixes are required together.
- The manifest stores the key as the literal string `process.env.NEXT_SERVER_ACTIONS_ENCRYPTION_KEY`, not the actual value. This is intentional — the key is read at runtime, so it must be present at both build time and runtime.
- The encryption key must be generated once, stored as a permanent CI/CD secret, and passed on every deployment. It must never be rotated unless you are prepared to invalidate all active user sessions simultaneously.
- In a real CI/CD pipeline, `GIT_HASH` should be set to `$(git rev-parse HEAD)`, tying each deployment's module IDs to a specific commit.
