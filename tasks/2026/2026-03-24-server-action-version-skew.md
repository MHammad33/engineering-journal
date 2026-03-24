# Task Name: Diagnosing "Failed to find Server Action" in Production Docker Deployments

> **Date:** 2026-03-24
> **Category:** Frontend / DevOps
> **Status:** Completed

---

## 1. The Challenge

**Context:**
A Next.js application deployed via Docker in production was intermittently throwing `Error: Failed to find Server Action "x". This request might be from an older or newer deployment`, but the error could not be reproduced locally in development.

**The "Wall":**
The error only surfaces under a specific combination of conditions: a user must have an old page open in the browser, and a new Docker deployment must have happened in the background. Because local dev mode skips encryption entirely and local Docker builds reuse build cache, the error was invisible in every non-production environmentmaking it extremely difficult to isolate and reproduce.

---

## 2. Constraints

**Technical:**
- Next.js `15.5.7` in production, `15.3.4` in the test project, different behavior between versions due to build cache and key generation differences
- Docker builds in production ran with `--no-cache`, invalidating Next.js build cache on every deployment
- No `NEXT_SERVER_ACTIONS_ENCRYPTION_KEY` set, meaning a new random encryption key was baked into every build

**Business:**
- Could not test directly against the production project, had to reproduce in an isolated new project
- Error was intermittent and user-facing, making it a silent UX failure with no visible client-side error boundary

---

## 3. The Decision Matrix (The "Thinking")

| Option | Pros | Cons | Decision |
| :--- | :--- | :--- | :--- |
| **Upgrade Next.js** | May include internal fixes | Doesn't address root cause, risk of regressions | Rejected as sole fix |
| **Set `NEXT_SERVER_ACTIONS_ENCRYPTION_KEY`** | Stable action IDs across all deployments, low effort, no code change | Key must be stored and managed as a long-lived secret | **Accepted** |
| **Force client refresh on deployment** | Eliminates stale clients entirely | Requires additional infrastructure (deployment ID header check, service worker) | Considered as complementary |

**Why setting `NEXT_SERVER_ACTIONS_ENCRYPTION_KEY`?**
It directly addresses the root cause — non-deterministic action ID and encryption key generation on every build, with a single environment variable, no code changes, and no risk to existing functionality.

---

## 4. Abstracted Implementation

```typescript
// Root cause: every build generates a new random encryptionKey
// baked into .next/server/server-reference-manifest.json
// Old client holds action ID from previous build → new server can't find it

// server-reference-manifest.json (without fix)
{
  "node": {
    "<action-id>": { ... }  // changes every build without cache
  },
  "encryptionKey": "<random-per-build>"  // new key every build
}

// Fix: set a stable key at build time
// Dockerfile
ARG NEXT_SERVER_ACTIONS_ENCRYPTION_KEY
ENV NEXT_SERVER_ACTIONS_ENCRYPTION_KEY $NEXT_SERVER_ACTIONS_ENCRYPTION_KEY

// Build command
docker build \
  --no-cache \
  --build-arg NEXT_SERVER_ACTIONS_ENCRYPTION_KEY=<stable-base64-32-byte-key> \
  -t myapp:latest .
```

---

## 5. Key Learnings

- `next dev` skips Server Action encryption entirely — this class of error is **impossible to hit locally in dev mode**
- Next.js build cache (valid for 14 days) keeps action IDs stable between local builds — masking the issue in non-production Docker tests
- `--no-cache` in Docker invalidates the Next.js build cache, which is what production CI pipelines effectively do on every deployment
- The fix requires generating the key **once**, storing it as a permanent CI/CD secret, and passing it as a build arg on every deployment — it must never be rotated unless you are prepared to invalidate all active user sessions
