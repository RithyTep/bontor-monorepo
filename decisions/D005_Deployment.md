# D005: Deployment Infrastructure

## Status
- [ ] Proposed
- [x] Approved
- [ ] Superseded by D###

## Context

Karaoke Cambodia needs deployment infrastructure for:
- Frontend (Next.js 16) hosting
- Backend API (tRPC) hosting
- Database hosting
- Media file storage and delivery
- CI/CD pipeline

The choice affects operational complexity, costs, and scalability.

## Alternatives Considered

### Option A: AWS (Full Stack)

**Pros:**
- Complete control over infrastructure
- Extensive service ecosystem
- Enterprise-grade reliability
- Global presence

**Cons:**
- High operational complexity
- Requires DevOps expertise
- Higher cost for small scale
- Over-engineering for MVP

### Option B: GCP (Firebase + Cloud Run)

**Pros:**
- Firebase for auth and hosting
- Cloud Run for serverless containers
- Good integration with Google services

**Cons:**
- Less Next.js optimized than Vercel
- More configuration needed
- Firebase limitations for complex apps

### Option C: Vercel + Neon + Cloudflare R2

**Pros:**
- Vercel: Purpose-built for Next.js
- Neon: Serverless PostgreSQL, auto-scaling
- R2: Cost-effective object storage (free egress)
- Minimal DevOps overhead
- Auto-scaling and edge deployment
- Generous free tiers

**Cons:**
- Vendor coupling (acceptable for MVP)
- Less control than self-hosted
- Vercel pricing at scale

## Decision

**Vercel + Neon + Cloudflare R2**

Deploy using:
- **Vercel**: Frontend (Next.js) and API (tRPC via Serverless Functions)
- **Neon**: PostgreSQL database
- **Cloudflare R2**: Audio and media storage

## Rationale

1. **Next.js Optimization:** Vercel is built by the Next.js team. RSC, streaming, edge middleware all work perfectly.

2. **Serverless Fit:** Both Vercel Functions and Neon are serverless, scaling to zero when idle and up under load.

3. **Developer Experience:** git push deploys, preview environments, instant rollbacks, built-in analytics.

4. **Cost Efficiency:** Free tiers cover development. Production costs are predictable and reasonable for MVP scale.

5. **R2 Economics:** Cloudflare R2's zero egress fees make audio streaming cost-effective.

6. **Cambodia Presence:** Cloudflare has POPs in Singapore/Bangkok, providing good latency for Cambodia users.

## Impact

### Architecture

```
┌─────────────────────────────────────────────────┐
│                    VERCEL                        │
│  ┌─────────────────────────────────────────┐    │
│  │  Edge Network (Global)                   │    │
│  │  - Static assets                         │    │
│  │  - Edge middleware (auth checks)         │    │
│  └─────────────────────────────────────────┘    │
│  ┌─────────────────────────────────────────┐    │
│  │  Serverless Functions (Regional)         │    │
│  │  - tRPC API routes                       │    │
│  │  - NextAuth endpoints                    │    │
│  └─────────────────────────────────────────┘    │
└─────────────────────────────────────────────────┘
           │                        │
           ▼                        ▼
┌─────────────────────┐   ┌─────────────────────┐
│       NEON          │   │   CLOUDFLARE R2     │
│  PostgreSQL         │   │  Audio/Media        │
│  - Auto-scaling     │   │  - CDN delivery     │
│  - Connection pool  │   │  - Zero egress      │
└─────────────────────┘   └─────────────────────┘
```

### Environment Configuration

```bash
# Vercel Project Settings
Framework: Next.js
Build Command: pnpm turbo build --filter=web
Output Directory: apps/web/.next
Install Command: pnpm install
Root Directory: /

# Environment Variables (Production)
DATABASE_URL=postgresql://...@neon.tech/...
NEXTAUTH_URL=https://karaoke.example.com
CLOUDFLARE_R2_ENDPOINT=https://...r2.cloudflarestorage.com
```

### CI/CD Pipeline

```yaml
# .github/workflows/deploy.yml
name: Deploy
on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: pnpm/action-setup@v2
      - uses: actions/setup-node@v4
      - run: pnpm install
      - run: pnpm lint
      - run: pnpm typecheck
      - run: pnpm test
      - run: pnpm build
      # Vercel handles actual deployment via GitHub integration
```

### Rollback Strategy

- Vercel: Instant rollback to any previous deployment
- Neon: Point-in-time recovery for database
- R2: Versioned audio URLs, no rollback needed

## Related

- Decisions: D004_CachingStrategy, D006_AudioStorage
- Implementation: All apps deployment
