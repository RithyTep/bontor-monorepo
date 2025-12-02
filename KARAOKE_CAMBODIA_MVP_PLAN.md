# Karaoke Cambodia MVP — Production-Ready Plan

> **Version:** 1.0.0
> **Last Updated:** 2024-12-02
> **Status:** Draft → Pending Approval
> **Owner:** Engineering Team

---

## Table of Contents

1. [Executive Summary](#1-executive-summary)
2. [Spec Kit Alignment](#2-spec-kit-alignment)
3. [System Architecture](#3-system-architecture)
4. [Monorepo & Folder Structure](#4-monorepo--folder-structure)
5. [Frontend — Next.js 16](#5-frontend--nextjs-16)
6. [Backend — tRPC & API](#6-backend--trpc--api)
7. [Database Design — PostgreSQL + Prisma](#7-database-design--postgresql--prisma)
8. [Karaoke Engine](#8-karaoke-engine)
9. [CMS / Admin](#9-cms--admin)
10. [MVP Roadmap](#10-mvp-roadmap)
11. [Non-functional Requirements](#11-non-functional-requirements)
12. [GitHub Spec Kit Integration](#12-github-spec-kit-integration)

---

## 1. Executive Summary

### 1.1 Product Vision

**Karaoke Cambodia** is a production-ready web karaoke platform optimized for Cambodia: fast song discovery, accurate Khmer lyrics sync, local audio hosting, pitch-based scoring, user favorites, and admin CMS for song management.

### 1.2 MVP Scope

| In Scope | Out of Scope (Post-MVP) |
|----------|------------------------|
| Song discovery & search (Khmer + English) | Live multiplayer rooms |
| Karaoke mode with mic input | Mobile native apps |
| Real-time lyrics sync | ML-based advanced scoring |
| Basic pitch scoring (0-100) | Offline PWA |
| User favorites & profiles | Paid subscriptions |
| Admin CMS for song management | Social sharing features |

### 1.3 Success Metrics

| Metric | Target | Measurement |
|--------|--------|-------------|
| Song page load time | ≤ 1s (cold) | Vercel Analytics |
| Search latency | ≤ 200ms | APM monitoring |
| Lyrics sync drift | ≤ 50ms | Test harness |
| Pitch scoring accuracy | ≥ 80% match | Labeled test set |
| API error rate | < 0.5% | Sentry/Datadog |
| 7-day retention | TBD by PM | Analytics |

---

## 2. Spec Kit Alignment

### 2.1 Decision Flow

```
PRD (Requirements)
       ↓
   Proposal (P###)
       ↓
   Decision (D###)
       ↓
   Feature Spec (F###)
       ↓
   Implementation (I###)
       ↓
   Code + Tests
```

### 2.2 Master Decision Log

| Decision ID | Context | Alternatives Considered | Decision | Rationale | Impact | Status |
|-------------|---------|------------------------|----------|-----------|--------|--------|
| D001 | Karaoke Engine Architecture | 1. Server-side scoring 2. Client-side scoring 3. Hybrid | **Client-side with optional server verification** | Lower latency, real-time feedback, reduced server costs | Core UX, scoring accuracy | Approved |
| D002 | Database Schema Design | 1. MongoDB 2. PostgreSQL 3. PlanetScale | **PostgreSQL (Neon)** | Relational integrity, pg_trgm for search, Prisma support | Data layer foundation | Approved |
| D003 | API Layer Architecture | 1. REST 2. GraphQL 3. tRPC | **tRPC** | Full type safety, monorepo synergy, faster development | FE/BE contract | Approved |
| D004 | Caching Strategy | 1. Redis 2. Vercel KV 3. In-memory + CDN | **CDN (Vercel Edge) + Browser Cache** | Cost-effective MVP, sufficient for launch scale | Performance | Approved |
| D005 | Deployment Infrastructure | 1. AWS 2. GCP 3. Vercel + Neon | **Vercel + Neon** | Serverless, auto-scaling, cost-effective MVP | DevOps simplicity | Approved |
| D006 | Audio Storage | 1. S3 2. Cloudflare R2 3. Vercel Blob | **Cloudflare R2** | Cost-effective egress, CDN integration | Media delivery | Approved |
| D007 | Authentication | 1. Custom JWT 2. Auth0 3. NextAuth | **NextAuth v5** | Built-in Next.js support, OAuth providers | User management | Approved |
| D008 | Pitch Detection Algorithm | 1. FFT-based 2. Autocorrelation 3. YIN | **YIN Algorithm** | Better accuracy for voice, proven in music apps | Scoring quality | Approved |
| D009 | Frontend Framework | 1. Next.js 14 2. Next.js 15 3. Next.js 16 | **Next.js 16** | Latest RSC, streaming, Turbopack | Performance + DX | Approved |
| D010 | State Management | 1. Redux 2. Zustand 3. Jotai | **Zustand** | Lightweight, TypeScript-first, minimal boilerplate | FE architecture | Approved |
| D011 | Lyrics Format | 1. Custom JSON 2. LRC 3. ASS/SSA | **LRC → JSON conversion** | Industry standard, easy parsing, compact storage | Content pipeline | Approved |
| D012 | Search Implementation | 1. Elasticsearch 2. Meilisearch 3. pg_trgm | **pg_trgm (MVP) → Meilisearch (scale)** | No extra infra for MVP, upgrade path clear | Search feature | Approved |

### 2.3 Feature Registry

| Feature ID | Feature Name | Decision Refs | Priority | Status |
|------------|--------------|---------------|----------|--------|
| F001 | Karaoke Mode | D001, D008 | P0 | In Progress |
| F002 | Song CMS | D002, D011 | P0 | Pending |
| F003 | Recording & Playback | D001, D006 | P1 | Pending |
| F004 | Scoring System | D001, D008 | P0 | In Progress |
| F005 | Admin Panel | D002, D007 | P1 | Pending |
| F006 | User Authentication | D007 | P0 | Pending |
| F007 | Song Search | D012 | P0 | Pending |
| F008 | Favorites | D002 | P1 | Pending |

---

## 3. System Architecture

### 3.1 High-Level Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              KARAOKE CAMBODIA                                │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  ┌────────────────┐    ┌────────────────┐    ┌────────────────────────────┐ │
│  │   CDN/Edge     │    │   CDN/Edge     │    │      CDN/Edge              │ │
│  │   (Vercel)     │    │ (Cloudflare)   │    │    (Cloudflare R2)         │ │
│  │   Static/SSR   │    │   DNS/WAF      │    │    Audio/Media             │ │
│  └───────┬────────┘    └───────┬────────┘    └───────────┬────────────────┘ │
│          │                     │                         │                   │
│          ▼                     ▼                         ▼                   │
│  ┌───────────────────────────────────────────────────────────────────────┐  │
│  │                        PRESENTATION LAYER                              │  │
│  │  ┌─────────────────────────────────────────────────────────────────┐  │  │
│  │  │                    Next.js 16 (App Router)                       │  │  │
│  │  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────────┐  │  │  │
│  │  │  │   RSC       │  │   Client    │  │    Karaoke Engine       │  │  │  │
│  │  │  │  (Server)   │  │ Components  │  │    (Web Audio API)      │  │  │  │
│  │  │  └─────────────┘  └─────────────┘  └─────────────────────────┘  │  │  │
│  │  └─────────────────────────────────────────────────────────────────┘  │  │
│  └───────────────────────────────────────────────────────────────────────┘  │
│                                    │                                         │
│                                    ▼                                         │
│  ┌───────────────────────────────────────────────────────────────────────┐  │
│  │                          API LAYER                                     │  │
│  │  ┌─────────────────────────────────────────────────────────────────┐  │  │
│  │  │                    tRPC Server (Vercel Serverless)               │  │  │
│  │  │  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────────────┐ │  │  │
│  │  │  │  Song    │  │  User    │  │  Auth    │  │  Score (opt.)    │ │  │  │
│  │  │  │  Router  │  │  Router  │  │  Router  │  │  Router          │ │  │  │
│  │  │  └──────────┘  └──────────┘  └──────────┘  └──────────────────┘ │  │  │
│  │  └─────────────────────────────────────────────────────────────────┘  │  │
│  └───────────────────────────────────────────────────────────────────────┘  │
│                                    │                                         │
│                                    ▼                                         │
│  ┌───────────────────────────────────────────────────────────────────────┐  │
│  │                          DATA LAYER                                    │  │
│  │  ┌─────────────────────────────────────────────────────────────────┐  │  │
│  │  │                    Prisma ORM                                    │  │  │
│  │  │  ┌──────────────────────────────────────────────────────────┐  │  │  │
│  │  │  │              Neon PostgreSQL (Serverless)                 │  │  │  │
│  │  │  │  ┌────────┐ ┌────────┐ ┌────────┐ ┌────────┐ ┌────────┐ │  │  │  │
│  │  │  │  │  User  │ │  Song  │ │ Artist │ │Category│ │Favorite│ │  │  │  │
│  │  │  │  └────────┘ └────────┘ └────────┘ └────────┘ └────────┘ │  │  │  │
│  │  │  └──────────────────────────────────────────────────────────┘  │  │  │
│  │  └─────────────────────────────────────────────────────────────────┘  │  │
│  └───────────────────────────────────────────────────────────────────────┘  │
│                                                                              │
│  ┌───────────────────────────────────────────────────────────────────────┐  │
│  │                       OBSERVABILITY                                    │  │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────────────────────┐ │  │
│  │  │   Sentry     │  │   Vercel     │  │      Structured Logging       │ │  │
│  │  │   (Errors)   │  │  Analytics   │  │       (JSON to stdout)        │ │  │
│  │  └──────────────┘  └──────────────┘  └──────────────────────────────┘ │  │
│  └───────────────────────────────────────────────────────────────────────┘  │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 3.2 FE/BE Boundaries

| Layer | Responsibility | Technology |
|-------|---------------|------------|
| **Presentation** | UI rendering, user interaction, karaoke engine | Next.js 16, React 19, Tailwind |
| **API Gateway** | Request routing, auth, rate limiting | Vercel Edge |
| **Application** | Business logic, data validation | tRPC routers, Zod |
| **Data Access** | DB queries, caching | Prisma, Neon |
| **Storage** | Media files, static assets | Cloudflare R2 |

### 3.3 Audio Processing Pipeline

```
┌─────────────────────────────────────────────────────────────────────────┐
│                     AUDIO PROCESSING PIPELINE                            │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  INGESTION (Admin CMS)                                                   │
│  ┌─────────┐    ┌─────────────┐    ┌─────────────┐    ┌──────────────┐  │
│  │  MP3    │───▶│  Normalize  │───▶│  Upload to  │───▶│  Store URL   │  │
│  │  Upload │    │  (ffmpeg)   │    │    R2       │    │   in DB      │  │
│  └─────────┘    └─────────────┘    └─────────────┘    └──────────────┘  │
│                                                                          │
│  ┌─────────┐    ┌─────────────┐    ┌─────────────┐    ┌──────────────┐  │
│  │  LRC    │───▶│  Parse to   │───▶│  Validate   │───▶│  Store JSON  │  │
│  │  Upload │    │    JSON     │    │  Timestamps │    │   in DB      │  │
│  └─────────┘    └─────────────┘    └─────────────┘    └──────────────┘  │
│                                                                          │
│  PLAYBACK (Client)                                                       │
│  ┌───────────────────────────────────────────────────────────────────┐  │
│  │                      Web Audio API Context                         │  │
│  │  ┌─────────────┐    ┌─────────────┐    ┌─────────────────────────┐│  │
│  │  │  Audio      │    │  Playback   │    │    Lyrics Sync          ││  │
│  │  │  Source     │───▶│  Controls   │───▶│    (RAF loop)           ││  │
│  │  └─────────────┘    └─────────────┘    └─────────────────────────┘│  │
│  │         │                                                          │  │
│  │         ▼                                                          │  │
│  │  ┌─────────────┐    ┌─────────────┐    ┌─────────────────────────┐│  │
│  │  │  Mic Input  │───▶│  YIN Pitch  │───▶│    Score Calculator     ││  │
│  │  │  (getUserMedia)   │  Detection  │    │    (frame comparison)   ││  │
│  │  └─────────────┘    └─────────────┘    └─────────────────────────┘│  │
│  └───────────────────────────────────────────────────────────────────┘  │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### 3.4 Deployment Architecture

```
┌─────────────────────────────────────────────────────────────────────────┐
│                       DEPLOYMENT ARCHITECTURE                            │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │                         VERCEL                                   │    │
│  │  ┌──────────────────┐  ┌──────────────────┐  ┌────────────────┐ │    │
│  │  │  Edge Functions  │  │  Serverless Fns  │  │  Static Assets │ │    │
│  │  │  (Middleware)    │  │  (tRPC/API)      │  │  (Next.js)     │ │    │
│  │  └──────────────────┘  └──────────────────┘  └────────────────┘ │    │
│  │                           │                                      │    │
│  └───────────────────────────│──────────────────────────────────────┘    │
│                              │                                           │
│                              ▼                                           │
│  ┌──────────────────┐   ┌─────────────────────────────────────────────┐ │
│  │   Cloudflare R2  │   │                    NEON                      │ │
│  │   (Audio CDN)    │   │              PostgreSQL                      │ │
│  │   ┌────────────┐ │   │   ┌────────────────────────────────────┐    │ │
│  │   │  MP3 files │ │   │   │  Serverless PostgreSQL             │    │ │
│  │   │  LRC files │ │   │   │  - Connection pooling              │    │ │
│  │   └────────────┘ │   │   │  - Auto-scaling                    │    │ │
│  └──────────────────┘   │   │  - Point-in-time recovery          │    │ │
│                         │   └────────────────────────────────────────┘ │ │
│                         └─────────────────────────────────────────────┘ │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 4. Monorepo & Folder Structure

### 4.1 Final Folder Structure

```
karaoke-cambodia/
├── apps/
│   ├── web/                          # Next.js 16 Frontend
│   │   ├── app/
│   │   │   ├── (public)/             # Public routes
│   │   │   │   ├── page.tsx          # Home
│   │   │   │   ├── songs/
│   │   │   │   │   ├── page.tsx      # Song listing
│   │   │   │   │   └── [id]/
│   │   │   │   │       └── page.tsx  # Song detail + karaoke
│   │   │   │   ├── search/
│   │   │   │   │   └── page.tsx
│   │   │   │   └── layout.tsx
│   │   │   ├── (auth)/               # Auth routes
│   │   │   │   ├── login/
│   │   │   │   ├── register/
│   │   │   │   └── layout.tsx
│   │   │   ├── (dashboard)/          # Protected routes
│   │   │   │   ├── profile/
│   │   │   │   ├── favorites/
│   │   │   │   └── layout.tsx
│   │   │   ├── api/
│   │   │   │   ├── trpc/[trpc]/
│   │   │   │   │   └── route.ts
│   │   │   │   └── auth/[...nextauth]/
│   │   │   │       └── route.ts
│   │   │   ├── layout.tsx
│   │   │   └── globals.css
│   │   ├── components/
│   │   │   ├── karaoke/
│   │   │   │   ├── KaraokeMode.tsx
│   │   │   │   ├── LyricsPanel.tsx
│   │   │   │   ├── PitchVisualizer.tsx
│   │   │   │   └── ScoreDisplay.tsx
│   │   │   ├── audio/
│   │   │   │   ├── AudioPlayer.tsx
│   │   │   │   └── AudioControls.tsx
│   │   │   ├── song/
│   │   │   │   ├── SongCard.tsx
│   │   │   │   ├── SongList.tsx
│   │   │   │   └── SongDetail.tsx
│   │   │   ├── search/
│   │   │   │   └── SearchBar.tsx
│   │   │   └── ui/                   # shadcn/ui components
│   │   ├── hooks/
│   │   │   ├── useKaraoke.ts
│   │   │   ├── usePitchDetection.ts
│   │   │   ├── useLyricsSync.ts
│   │   │   └── useAudio.ts
│   │   ├── lib/
│   │   │   ├── trpc.ts               # tRPC client
│   │   │   ├── auth.ts               # NextAuth config
│   │   │   └── utils.ts
│   │   ├── stores/
│   │   │   ├── karaoke-store.ts
│   │   │   └── user-store.ts
│   │   ├── next.config.js
│   │   ├── tailwind.config.js
│   │   └── package.json
│   │
│   └── admin/                        # Admin Panel (Next.js)
│       ├── app/
│       │   ├── dashboard/
│       │   ├── songs/
│       │   │   ├── page.tsx
│       │   │   ├── new/
│       │   │   └── [id]/edit/
│       │   ├── artists/
│       │   ├── categories/
│       │   └── layout.tsx
│       ├── components/
│       └── package.json
│
├── packages/
│   ├── db/                           # Prisma + Database
│   │   ├── prisma/
│   │   │   ├── schema.prisma
│   │   │   ├── migrations/
│   │   │   └── seed.ts
│   │   ├── client.ts
│   │   └── package.json
│   │
│   ├── trpc/                         # Shared tRPC
│   │   ├── routers/
│   │   │   ├── song.ts
│   │   │   ├── user.ts
│   │   │   ├── auth.ts
│   │   │   ├── favorite.ts
│   │   │   ├── artist.ts
│   │   │   ├── category.ts
│   │   │   └── index.ts
│   │   ├── context.ts
│   │   ├── middleware.ts
│   │   ├── trpc.ts
│   │   └── package.json
│   │
│   ├── types/                        # Shared Types
│   │   ├── song.ts
│   │   ├── user.ts
│   │   ├── lyrics.ts
│   │   ├── scoring.ts
│   │   ├── index.ts
│   │   └── package.json
│   │
│   ├── utils/                        # Shared Utilities
│   │   ├── lrc-parser.ts
│   │   ├── pitch-detection.ts
│   │   ├── scoring.ts
│   │   ├── audio-utils.ts
│   │   ├── validators.ts
│   │   └── package.json
│   │
│   ├── ui/                           # Shared UI Components
│   │   ├── components/
│   │   │   ├── button.tsx
│   │   │   ├── card.tsx
│   │   │   ├── input.tsx
│   │   │   ├── modal.tsx
│   │   │   └── toast.tsx
│   │   ├── index.ts
│   │   └── package.json
│   │
│   ├── karaoke-engine/               # Audio Engine Package
│   │   ├── src/
│   │   │   ├── pitch-detector.ts
│   │   │   ├── lyrics-sync.ts
│   │   │   ├── score-calculator.ts
│   │   │   ├── audio-recorder.ts
│   │   │   └── index.ts
│   │   └── package.json
│   │
│   └── config/                       # Shared Configs
│       ├── eslint/
│       │   └── index.js
│       ├── typescript/
│       │   └── base.json
│       └── tailwind/
│           └── tailwind.config.js
│
├── decisions/                        # Spec Kit: Decision Logs
│   ├── D001_KaraokeEngine.md
│   ├── D002_DBSchema.md
│   ├── D003_APIEndpoints.md
│   ├── D004_CachingStrategy.md
│   ├── D005_Deployment.md
│   ├── D006_AudioStorage.md
│   ├── D007_Authentication.md
│   ├── D008_PitchDetection.md
│   ├── D009_FrontendFramework.md
│   ├── D010_StateManagement.md
│   ├── D011_LyricsFormat.md
│   └── D012_SearchImplementation.md
│
├── features/                         # Spec Kit: Feature Specs
│   ├── F001_KaraokeMode.md
│   ├── F002_SongCMS.md
│   ├── F003_Recording.md
│   ├── F004_Scoring.md
│   ├── F005_AdminPanel.md
│   ├── F006_Authentication.md
│   ├── F007_Search.md
│   └── F008_Favorites.md
│
├── proposals/                        # Spec Kit: Proposals
│   ├── P001_AudioProcessingPipeline.md
│   ├── P002_RealTimeLyrics.md
│   └── P003_DBIndexing.md
│
├── implementation/                   # Spec Kit: Implementation Notes
│   ├── I001_FrontendStructure.md
│   ├── I002_BackendAPI.md
│   ├── I003_CMSWorkflow.md
│   └── I004_KaraokeEngine.md
│
├── scripts/                          # DevOps Scripts
│   ├── dev.sh
│   ├── build.sh
│   ├── deploy.sh
│   ├── seed-db.sh
│   └── migrate.sh
│
├── .github/
│   ├── workflows/
│   │   ├── ci.yml
│   │   ├── deploy-preview.yml
│   │   └── deploy-production.yml
│   ├── PULL_REQUEST_TEMPLATE.md
│   └── CODEOWNERS
│
├── .gitattributes                    # Git LFS config
├── .gitignore
├── .env.example
├── package.json
├── pnpm-workspace.yaml
├── turbo.json
├── tsconfig.json
└── README.md
```

### 4.2 Turborepo Configuration

**`turbo.json`**:
```json
{
  "$schema": "https://turbo.build/schema.json",
  "globalDependencies": ["**/.env.*local"],
  "pipeline": {
    "build": {
      "dependsOn": ["^build"],
      "outputs": [".next/**", "!.next/cache/**", "dist/**"]
    },
    "dev": {
      "cache": false,
      "persistent": true
    },
    "lint": {
      "dependsOn": ["^build"]
    },
    "test": {
      "dependsOn": ["^build"]
    },
    "typecheck": {
      "dependsOn": ["^build"]
    },
    "db:generate": {
      "cache": false
    },
    "db:migrate": {
      "cache": false
    },
    "db:seed": {
      "cache": false
    }
  }
}
```

**`pnpm-workspace.yaml`**:
```yaml
packages:
  - 'apps/*'
  - 'packages/*'
```

### 4.3 TypeScript Paths Configuration

**Root `tsconfig.json`**:
```json
{
  "compilerOptions": {
    "baseUrl": ".",
    "paths": {
      "@karaoke/db": ["packages/db"],
      "@karaoke/trpc": ["packages/trpc"],
      "@karaoke/types": ["packages/types"],
      "@karaoke/utils": ["packages/utils"],
      "@karaoke/ui": ["packages/ui"],
      "@karaoke/karaoke-engine": ["packages/karaoke-engine"],
      "@karaoke/config/*": ["packages/config/*"]
    },
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "forceConsistentCasingInFileNames": true,
    "moduleResolution": "bundler",
    "resolveJsonModule": true,
    "isolatedModules": true,
    "noEmit": true
  }
}
```

### 4.4 Git Workflow & Branch Strategy

```
main              ← Production (protected)
  │
  ├── develop     ← Integration branch
  │     │
  │     ├── feature/F001-karaoke-mode
  │     ├── feature/F002-song-cms
  │     ├── feature/F003-recording
  │     └── ...
  │
  ├── hotfix/critical-bug-fix
  │
  └── release/v1.0.0
```

### 4.5 Commit Conventions

```
<type>(<scope>): <description>

[optional body]

[optional footer]
```

**Types:**
| Type | Description |
|------|-------------|
| `feat` | New feature |
| `fix` | Bug fix |
| `docs` | Documentation only |
| `style` | Code style (formatting, etc.) |
| `refactor` | Code refactoring |
| `perf` | Performance improvement |
| `test` | Adding tests |
| `chore` | Build/config changes |

**Scopes:**
- `web`, `admin`, `api`, `db`, `trpc`, `utils`, `ui`, `karaoke-engine`
- `decisions`, `features`, `proposals`, `implementation`

**Examples:**
```
feat(karaoke-engine): Add YIN pitch detection algorithm
fix(web): Fix lyrics sync drift on seek
docs(decisions): Add D008_PitchDetection decision log
refactor(trpc): Restructure song router for pagination
```

### 4.6 PR Template

**`.github/PULL_REQUEST_TEMPLATE.md`**:
```markdown
## Summary

<!-- Brief description of changes -->

## Related Items

- Decision: D###_DecisionName
- Feature: F###_FeatureName
- Issue: #123

## Type of Change

- [ ] Feature (new functionality)
- [ ] Bug fix
- [ ] Refactor
- [ ] Documentation
- [ ] Config/Build

## Checklist

- [ ] Code follows project style guidelines
- [ ] Self-review completed
- [ ] Tests added/updated
- [ ] Documentation updated
- [ ] Decision log updated (if applicable)
- [ ] No console.log or debug code

## Screenshots (if applicable)

<!-- Add screenshots for UI changes -->

## Testing Instructions

<!-- How to test this PR -->
```

---

## 5. Frontend — Next.js 16

### 5.1 Component Architecture Decision Table

| Decision ID | Component | Decision | Rationale |
|-------------|-----------|----------|-----------|
| FE-001 | KaraokeMode | Client Component | Requires Web Audio API, mic access, RAF loops |
| FE-002 | SongList | RSC with streaming | Data fetching on server, streaming for fast FCP |
| FE-003 | SearchBar | Client Component | Real-time input, debounced API calls |
| FE-004 | SongCard | RSC | Static content, no interactivity |
| FE-005 | AudioPlayer | Client Component | Audio element controls, state management |
| FE-006 | LyricsPanel | Client Component | Real-time sync with audio |
| FE-007 | PitchVisualizer | Client Component | Canvas/WebGL rendering |
| FE-008 | Navigation | RSC | Static navigation links |
| FE-009 | UserMenu | Client Component | Auth state, dropdown interaction |
| FE-010 | FavoriteButton | Client Component | Optimistic updates, user state |

### 5.2 Routing Strategy

```
app/
├── (public)/                    # Public layout group
│   ├── layout.tsx              # Public layout (navbar, footer)
│   ├── page.tsx                # Home: RSC, streaming song cards
│   ├── songs/
│   │   ├── page.tsx            # Song list: RSC + Suspense
│   │   └── [id]/
│   │       └── page.tsx        # Song detail: RSC shell + Client karaoke
│   └── search/
│       └── page.tsx            # Search: RSC + Client SearchBar
│
├── (auth)/                      # Auth layout group
│   ├── layout.tsx              # Minimal auth layout
│   ├── login/
│   │   └── page.tsx            # Client: OAuth buttons
│   └── register/
│       └── page.tsx
│
├── (dashboard)/                 # Protected layout group
│   ├── layout.tsx              # Auth check, redirect if not logged in
│   ├── profile/
│   │   └── page.tsx            # User profile: RSC
│   └── favorites/
│       └── page.tsx            # Favorites list: RSC + streaming
│
└── api/
    ├── trpc/[trpc]/
    │   └── route.ts            # tRPC handler
    └── auth/[...nextauth]/
        └── route.ts            # NextAuth handler
```

### 5.3 Suspense & Streaming Pattern

```tsx
// app/(public)/songs/page.tsx
import { Suspense } from 'react'
import { SongListSkeleton } from '@/components/song/SongListSkeleton'
import { SongList } from '@/components/song/SongList'

export default function SongsPage() {
  return (
    <main className="container mx-auto py-8">
      <h1 className="text-3xl font-bold mb-6">Browse Songs</h1>
      <Suspense fallback={<SongListSkeleton />}>
        <SongList />
      </Suspense>
    </main>
  )
}
```

### 5.4 Offline & Caching Strategy

| Resource | Strategy | TTL |
|----------|----------|-----|
| Static pages | stale-while-revalidate | 1 hour |
| Song metadata | cache, revalidate on focus | 5 min |
| Audio files | immutable cache | Forever (versioned URLs) |
| Lyrics JSON | cache | 1 hour |
| User data | no-store | N/A |

### 5.5 SEO & Metadata

```tsx
// app/(public)/songs/[id]/page.tsx
import { Metadata } from 'next'

export async function generateMetadata({ params }): Promise<Metadata> {
  const song = await getSong(params.id)

  return {
    title: `${song.title} - ${song.artistName} | Karaoke Cambodia`,
    description: `Sing ${song.title} by ${song.artistName} with synchronized lyrics and pitch scoring.`,
    openGraph: {
      title: song.title,
      description: `Karaoke: ${song.title}`,
      images: [song.coverUrl],
    },
  }
}
```

---

## 6. Backend — tRPC & API

### 6.1 Router Organization

```typescript
// packages/trpc/routers/index.ts
import { router } from '../trpc'
import { songRouter } from './song'
import { userRouter } from './user'
import { authRouter } from './auth'
import { favoriteRouter } from './favorite'
import { artistRouter } from './artist'
import { categoryRouter } from './category'
import { scoreRouter } from './score'

export const appRouter = router({
  song: songRouter,
  user: userRouter,
  auth: authRouter,
  favorite: favoriteRouter,
  artist: artistRouter,
  category: categoryRouter,
  score: scoreRouter,
})

export type AppRouter = typeof appRouter
```

### 6.2 Router Design Decisions

| Router | Endpoints | Auth Required | Rate Limit |
|--------|-----------|---------------|------------|
| song | list, get, search | No | 100/min |
| user | me, update | Yes | 30/min |
| auth | session, signOut | Varies | 10/min |
| favorite | list, add, remove | Yes | 30/min |
| artist | list, get | No | 100/min |
| category | list | No | 100/min |
| score | submit, history | Yes | 20/min |

### 6.3 Input/Output Validation (Zod)

```typescript
// packages/trpc/routers/song.ts
import { z } from 'zod'
import { router, publicProcedure, protectedProcedure } from '../trpc'
import { prisma } from '@karaoke/db'

const SongListInput = z.object({
  q: z.string().optional(),
  categoryId: z.string().uuid().optional(),
  skip: z.number().int().min(0).default(0),
  take: z.number().int().min(1).max(50).default(20),
})

const SongGetInput = z.object({
  id: z.string().uuid(),
})

export const songRouter = router({
  list: publicProcedure
    .input(SongListInput)
    .query(async ({ input }) => {
      const { q, categoryId, skip, take } = input

      const where = {
        isPublished: true,
        ...(categoryId && { categoryId }),
        ...(q && {
          OR: [
            { title: { contains: q, mode: 'insensitive' } },
            { artist: { name: { contains: q, mode: 'insensitive' } } },
          ],
        }),
      }

      const [items, total] = await Promise.all([
        prisma.song.findMany({
          where,
          skip,
          take,
          include: { artist: true, category: true },
          orderBy: { createdAt: 'desc' },
        }),
        prisma.song.count({ where }),
      ])

      return { items, total }
    }),

  get: publicProcedure
    .input(SongGetInput)
    .query(async ({ input }) => {
      const song = await prisma.song.findUnique({
        where: { id: input.id, isPublished: true },
        include: { artist: true, category: true },
      })

      if (!song) {
        throw new TRPCError({ code: 'NOT_FOUND' })
      }

      return song
    }),
})
```

### 6.4 Service Layer Pattern

```typescript
// apps/api/src/services/song-service.ts
import { prisma } from '@karaoke/db'
import type { SongDTO } from '@karaoke/types'

export class SongService {
  static async findById(id: string): Promise<SongDTO | null> {
    const song = await prisma.song.findUnique({
      where: { id },
      include: { artist: true, category: true },
    })

    if (!song) return null

    return this.toDTO(song)
  }

  static async search(query: string, options: SearchOptions): Promise<SearchResult> {
    // Implementation using pg_trgm
  }

  private static toDTO(song: SongWithRelations): SongDTO {
    return {
      id: song.id,
      title: song.title,
      artistId: song.artistId,
      artistName: song.artist.name,
      categoryId: song.categoryId,
      audioUrl: song.audioUrl,
      lyricsJson: song.lyricsJson as LyricsLine[],
      durationSec: song.durationSec,
      isPublished: song.isPublished,
    }
  }
}
```

### 6.5 Error Handling

```typescript
// packages/trpc/middleware.ts
import { TRPCError } from '@trpc/server'

export const errorHandler = t.middleware(async ({ next }) => {
  try {
    return await next()
  } catch (error) {
    if (error instanceof TRPCError) throw error

    // Log to Sentry
    Sentry.captureException(error)

    throw new TRPCError({
      code: 'INTERNAL_SERVER_ERROR',
      message: 'An unexpected error occurred',
    })
  }
})
```

---

## 7. Database Design — PostgreSQL + Prisma

### 7.1 Complete Prisma Schema

```prisma
// packages/db/prisma/schema.prisma

generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

// ============================================
// USER & AUTH
// ============================================

model User {
  id            String    @id @default(uuid())
  email         String    @unique
  emailVerified DateTime?
  name          String?
  image         String?

  // Relations
  accounts      Account[]
  sessions      Session[]
  favorites     Favorite[]
  scores        Score[]

  createdAt     DateTime  @default(now())
  updatedAt     DateTime  @updatedAt

  @@index([email])
}

model Account {
  id                String  @id @default(uuid())
  userId            String
  type              String
  provider          String
  providerAccountId String
  refresh_token     String? @db.Text
  access_token      String? @db.Text
  expires_at        Int?
  token_type        String?
  scope             String?
  id_token          String? @db.Text
  session_state     String?

  user User @relation(fields: [userId], references: [id], onDelete: Cascade)

  @@unique([provider, providerAccountId])
  @@index([userId])
}

model Session {
  id           String   @id @default(uuid())
  sessionToken String   @unique
  userId       String
  expires      DateTime

  user         User     @relation(fields: [userId], references: [id], onDelete: Cascade)

  @@index([userId])
}

model VerificationToken {
  identifier String
  token      String   @unique
  expires    DateTime

  @@unique([identifier, token])
}

// ============================================
// CONTENT
// ============================================

model Artist {
  id        String   @id @default(uuid())
  name      String
  slug      String   @unique
  imageUrl  String?
  bio       String?  @db.Text

  songs     Song[]

  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt

  @@index([slug])
  @@index([name])
}

model Category {
  id        String   @id @default(uuid())
  name      String
  slug      String   @unique

  songs     Song[]

  createdAt DateTime @default(now())

  @@index([slug])
}

model Song {
  id          String   @id @default(uuid())
  title       String
  slug        String   @unique

  // Relations
  artistId    String
  artist      Artist   @relation(fields: [artistId], references: [id])
  categoryId  String
  category    Category @relation(fields: [categoryId], references: [id])

  // Media
  audioUrl    String
  lrcUrl      String?
  lyricsJson  Json     // [{t: number, l: string}]
  coverUrl    String?

  // Metadata
  durationSec Int
  language    String?  // "km", "en", "th", "zh"
  bpm         Int?
  key         String?  // "C", "Am", etc.

  // Reference pitch data (for scoring)
  pitchData   Json?    // [{t: number, hz: number}]

  // Status
  isPublished Boolean  @default(false)
  publishedAt DateTime?

  // Stats
  playCount   Int      @default(0)

  // Relations
  favorites   Favorite[]
  scores      Score[]

  createdAt   DateTime @default(now())
  updatedAt   DateTime @updatedAt

  @@index([artistId])
  @@index([categoryId])
  @@index([isPublished])
  @@index([title])
  @@index([slug])
}

// ============================================
// USER ACTIVITY
// ============================================

model Favorite {
  id        String   @id @default(uuid())

  userId    String
  user      User     @relation(fields: [userId], references: [id], onDelete: Cascade)

  songId    String
  song      Song     @relation(fields: [songId], references: [id], onDelete: Cascade)

  createdAt DateTime @default(now())

  @@unique([userId, songId])
  @@index([userId])
  @@index([songId])
}

model Score {
  id              String   @id @default(uuid())

  userId          String
  user            User     @relation(fields: [userId], references: [id], onDelete: Cascade)

  songId          String
  song            Song     @relation(fields: [songId], references: [id], onDelete: Cascade)

  // Scoring breakdown
  totalScore      Int      // 0-100
  pitchAccuracy   Float    // 0.0-1.0
  durationCoverage Float   // 0.0-1.0

  // Recording (optional)
  recordingUrl    String?

  // Session metadata
  transposeSemitones Int   @default(0)
  toleranceCents     Int   @default(80)

  createdAt       DateTime @default(now())

  @@index([userId])
  @@index([songId])
  @@index([totalScore])
}
```

### 7.2 Indexing Strategy

| Table | Index | Type | Purpose |
|-------|-------|------|---------|
| song | `title` | B-tree | Exact match queries |
| song | `(title) gin_trgm_ops` | GIN | Fuzzy search |
| song | `artistId` | B-tree | Join performance |
| song | `categoryId` | B-tree | Filter by category |
| song | `isPublished` | B-tree | Filter published |
| user | `email` | B-tree | Login lookup |
| favorite | `(userId, songId)` | Unique | Deduplication |
| score | `totalScore DESC` | B-tree | Leaderboard queries |

### 7.3 SQL for pg_trgm Search

```sql
-- Enable extension
CREATE EXTENSION IF NOT EXISTS pg_trgm;

-- Create GIN index for fuzzy search
CREATE INDEX song_title_trgm_idx ON "Song" USING gin (title gin_trgm_ops);
CREATE INDEX artist_name_trgm_idx ON "Artist" USING gin (name gin_trgm_ops);

-- Example search query
SELECT s.*, similarity(s.title, 'search term') as sim
FROM "Song" s
WHERE s.title % 'search term'
ORDER BY sim DESC
LIMIT 20;
```

### 7.4 Migration Strategy

1. **Development**: `pnpm db:migrate dev` (creates migration files)
2. **Staging/Production**: `pnpm db:migrate deploy` (applies migrations)
3. **Rollback**: Maintain reversible migrations, test on staging first

---

## 8. Karaoke Engine

### 8.1 Engine Architecture

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        KARAOKE ENGINE PACKAGE                            │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │                      KaraokeEngine (Main Class)                  │    │
│  │                                                                   │    │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────────┐  │    │
│  │  │ AudioSource │  │ LyricsSync  │  │    PitchDetector        │  │    │
│  │  │             │  │             │  │    (YIN Algorithm)      │  │    │
│  │  │ - play()    │  │ - sync()    │  │                         │  │    │
│  │  │ - pause()   │  │ - seek()    │  │ - detect(buffer)        │  │    │
│  │  │ - seek()    │  │ - highlight │  │ - getFrequency()        │  │    │
│  │  └──────┬──────┘  └──────┬──────┘  └───────────┬─────────────┘  │    │
│  │         │                │                      │                │    │
│  │         ▼                ▼                      ▼                │    │
│  │  ┌───────────────────────────────────────────────────────────┐  │    │
│  │  │                    ScoreCalculator                         │  │    │
│  │  │                                                            │  │    │
│  │  │  - calculateFrameScore(detected, reference)                │  │    │
│  │  │  - calculateFinalScore()                                   │  │    │
│  │  │  - getBreakdown()                                          │  │    │
│  │  └───────────────────────────────────────────────────────────┘  │    │
│  │                              │                                   │    │
│  │                              ▼                                   │    │
│  │  ┌───────────────────────────────────────────────────────────┐  │    │
│  │  │                    AudioRecorder                           │  │    │
│  │  │                                                            │  │    │
│  │  │  - startRecording()                                        │  │    │
│  │  │  - stopRecording()                                         │  │    │
│  │  │  - getBlob()                                               │  │    │
│  │  └───────────────────────────────────────────────────────────┘  │    │
│  └─────────────────────────────────────────────────────────────────┘    │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### 8.2 Lyrics Sync Algorithm

```typescript
// packages/karaoke-engine/src/lyrics-sync.ts

export class LyricsSync {
  private lyrics: LyricsLine[]
  private currentIndex: number = 0
  private rafId: number | null = null
  private onHighlight: (index: number) => void

  constructor(lyrics: LyricsLine[], onHighlight: (index: number) => void) {
    this.lyrics = lyrics
    this.onHighlight = onHighlight
  }

  start(audioElement: HTMLAudioElement): void {
    const loop = () => {
      const currentTime = audioElement.currentTime
      this.updateHighlight(currentTime)
      this.rafId = requestAnimationFrame(loop)
    }
    this.rafId = requestAnimationFrame(loop)
  }

  stop(): void {
    if (this.rafId !== null) {
      cancelAnimationFrame(this.rafId)
      this.rafId = null
    }
  }

  seek(time: number): void {
    // Binary search for efficient seeking
    this.currentIndex = this.binarySearch(time)
  }

  private updateHighlight(time: number): void {
    const n = this.lyrics.length
    if (n === 0) return

    // Advance index if next lyric's timestamp has passed
    while (
      this.currentIndex + 1 < n &&
      this.lyrics[this.currentIndex + 1].t <= time + 0.05
    ) {
      this.currentIndex++
    }

    // Rewind if current time is before current lyric
    while (
      this.currentIndex > 0 &&
      this.lyrics[this.currentIndex].t > time + 0.05
    ) {
      this.currentIndex--
    }

    this.onHighlight(this.currentIndex)
  }

  private binarySearch(time: number): number {
    let lo = 0
    let hi = this.lyrics.length - 1

    while (lo <= hi) {
      const mid = Math.floor((lo + hi) / 2)
      if (this.lyrics[mid].t === time) return mid
      if (this.lyrics[mid].t < time) lo = mid + 1
      else hi = mid - 1
    }

    return Math.max(0, lo - 1)
  }
}
```

### 8.3 YIN Pitch Detection

```typescript
// packages/karaoke-engine/src/pitch-detector.ts

export class PitchDetector {
  private sampleRate: number
  private bufferSize: number
  private threshold: number = 0.15

  constructor(sampleRate: number, bufferSize: number = 2048) {
    this.sampleRate = sampleRate
    this.bufferSize = bufferSize
  }

  detect(buffer: Float32Array): number {
    const yinBuffer = this.calculateYIN(buffer)
    const tau = this.findTau(yinBuffer)

    if (tau === -1) return -1

    // Parabolic interpolation for better accuracy
    const refinedTau = this.parabolicInterpolation(yinBuffer, tau)
    const frequency = this.sampleRate / refinedTau

    // Filter out unrealistic frequencies
    if (frequency < 50 || frequency > 2000) return -1

    return frequency
  }

  private calculateYIN(buffer: Float32Array): Float32Array {
    const halfSize = Math.floor(buffer.length / 2)
    const yinBuffer = new Float32Array(halfSize)

    // Step 1: Difference function
    for (let tau = 0; tau < halfSize; tau++) {
      let sum = 0
      for (let i = 0; i < halfSize; i++) {
        const diff = buffer[i] - buffer[i + tau]
        sum += diff * diff
      }
      yinBuffer[tau] = sum
    }

    // Step 2: Cumulative mean normalized difference
    yinBuffer[0] = 1
    let runningSum = 0
    for (let tau = 1; tau < halfSize; tau++) {
      runningSum += yinBuffer[tau]
      yinBuffer[tau] = yinBuffer[tau] * tau / runningSum
    }

    return yinBuffer
  }

  private findTau(yinBuffer: Float32Array): number {
    for (let tau = 2; tau < yinBuffer.length; tau++) {
      if (yinBuffer[tau] < this.threshold) {
        // Find local minimum
        while (
          tau + 1 < yinBuffer.length &&
          yinBuffer[tau + 1] < yinBuffer[tau]
        ) {
          tau++
        }
        return tau
      }
    }
    return -1
  }

  private parabolicInterpolation(yinBuffer: Float32Array, tau: number): number {
    const x0 = tau > 0 ? yinBuffer[tau - 1] : yinBuffer[tau]
    const x1 = yinBuffer[tau]
    const x2 = tau + 1 < yinBuffer.length ? yinBuffer[tau + 1] : yinBuffer[tau]

    const a = (x0 + x2 - 2 * x1) / 2
    const b = (x2 - x0) / 2

    return a !== 0 ? tau - b / (2 * a) : tau
  }
}
```

### 8.4 Scoring System

```typescript
// packages/karaoke-engine/src/score-calculator.ts

export interface ScoreBreakdown {
  totalScore: number        // 0-100
  pitchAccuracy: number     // 0.0-1.0
  durationCoverage: number  // 0.0-1.0
  correctFrames: number
  totalFrames: number
}

export class ScoreCalculator {
  private correctFrames: number = 0
  private totalFrames: number = 0
  private toleranceCents: number

  constructor(toleranceCents: number = 80) {
    this.toleranceCents = toleranceCents
  }

  addFrame(detectedHz: number | null, referenceHz: number | null): void {
    this.totalFrames++

    if (detectedHz === null || referenceHz === null) {
      return // No pitch detected or no reference
    }

    const centsDiff = Math.abs(this.hzToCents(detectedHz, referenceHz))

    if (centsDiff <= this.toleranceCents) {
      this.correctFrames++
    }
  }

  reset(): void {
    this.correctFrames = 0
    this.totalFrames = 0
  }

  getBreakdown(): ScoreBreakdown {
    const pitchAccuracy = this.totalFrames > 0
      ? this.correctFrames / this.totalFrames
      : 0

    // Duration coverage could be based on frames with detected pitch
    const durationCoverage = pitchAccuracy // Simplified for MVP

    const totalScore = Math.round(
      (pitchAccuracy * 0.7 + durationCoverage * 0.3) * 100
    )

    return {
      totalScore,
      pitchAccuracy,
      durationCoverage,
      correctFrames: this.correctFrames,
      totalFrames: this.totalFrames,
    }
  }

  private hzToCents(hz: number, refHz: number): number {
    return 1200 * Math.log2(hz / refHz)
  }
}
```

### 8.5 Engine Decision Log

| Decision ID | Context | Decision | Rationale |
|-------------|---------|----------|-----------|
| KE-001 | Pitch algorithm | YIN | Better for voice, proven accuracy |
| KE-002 | Analysis frame size | 2048 samples | Balance accuracy/latency |
| KE-003 | Sync method | RAF loop | Smoother than setInterval |
| KE-004 | Scoring tolerance | 80 cents default | User-adjustable |
| KE-005 | Reference pitch | Server-provided JSON | Precomputed for accuracy |

---

## 9. CMS / Admin

### 9.1 Admin Features

| Feature | Description | Priority |
|---------|-------------|----------|
| Song CRUD | Create, read, update, delete songs | P0 |
| Audio upload | MP3 upload to R2, normalize | P0 |
| LRC upload | Parse LRC, validate timestamps | P0 |
| Artist management | CRUD artists | P1 |
| Category management | CRUD categories | P1 |
| Publish control | Toggle song visibility | P0 |
| Dashboard | Stats overview | P2 |

### 9.2 Song Ingestion Workflow

```
┌─────────────────────────────────────────────────────────────────────────┐
│                     SONG INGESTION WORKFLOW                              │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  1. ADMIN UPLOAD                                                         │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │  Admin selects:                                                  │    │
│  │  - MP3 file                                                      │    │
│  │  - LRC file                                                      │    │
│  │  - Metadata (title, artist, category)                           │    │
│  └─────────────────────────────────────────────────────────────────┘    │
│                              │                                           │
│                              ▼                                           │
│  2. VALIDATION                                                           │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │  - Validate file types (MP3, LRC)                               │    │
│  │  - Check file sizes (max 50MB audio)                            │    │
│  │  - Parse LRC, validate timestamp format                         │    │
│  └─────────────────────────────────────────────────────────────────┘    │
│                              │                                           │
│                              ▼                                           │
│  3. PROCESSING                                                           │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │  - Upload MP3 to Cloudflare R2                                  │    │
│  │  - Upload LRC to R2 (backup)                                    │    │
│  │  - Convert LRC → JSON                                           │    │
│  │  - Extract audio duration                                        │    │
│  │  - Generate slug from title                                      │    │
│  └─────────────────────────────────────────────────────────────────┘    │
│                              │                                           │
│                              ▼                                           │
│  4. DATABASE                                                             │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │  - Create Song record in DB                                     │    │
│  │  - Store audioUrl, lyricsJson, metadata                         │    │
│  │  - Set isPublished = false (draft)                              │    │
│  └─────────────────────────────────────────────────────────────────┘    │
│                              │                                           │
│                              ▼                                           │
│  5. REVIEW & PUBLISH                                                     │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │  - Admin previews song in admin panel                           │    │
│  │  - Admin can edit lyrics timing                                 │    │
│  │  - Admin clicks "Publish"                                       │    │
│  │  - isPublished = true, publishedAt = now()                      │    │
│  └─────────────────────────────────────────────────────────────────┘    │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### 9.3 Lyrics Editor

**Features:**
- Visual timeline with waveform
- Click to set timestamp for each line
- Preview sync in real-time
- Adjust timestamps with arrow keys (±10ms)
- Auto-save drafts

### 9.4 Admin UI Components

| Component | Description |
|-----------|-------------|
| SongForm | Title, artist, category, audio upload, LRC upload |
| LyricsEditor | Timeline-based lyrics timestamp editor |
| AudioUploader | Drag-drop MP3, progress indicator |
| SongTable | Sortable, filterable song list with actions |
| PublishToggle | Quick publish/unpublish button |

---

## 10. MVP Roadmap

### 10.1 Phase Overview

| Phase | Duration | Focus |
|-------|----------|-------|
| Phase 1: Foundation | Weeks 1-4 | Monorepo, DB, Auth, Core UI |
| Phase 2: Core Features | Weeks 5-8 | Karaoke engine, Song playback, Scoring |
| Phase 3: Admin & CMS | Weeks 9-10 | Admin panel, Song ingestion |
| Phase 4: Polish & Launch | Weeks 11-12 | Testing, Performance, Deployment |

### 10.2 Detailed Weekly Milestones

#### Phase 1: Foundation (Weeks 1-4)

**Week 1: Project Setup**
- [ ] Initialize monorepo with Turborepo
- [ ] Configure pnpm workspaces
- [ ] Set up TypeScript paths
- [ ] Configure ESLint + Prettier
- [ ] Create shared packages (types, utils, ui)
- [ ] Initialize Next.js 16 apps (web, admin)
- [ ] Set up Prisma with Neon

**Week 2: Database & Auth**
- [ ] Finalize Prisma schema
- [ ] Create initial migration
- [ ] Implement NextAuth v5
- [ ] Google OAuth integration
- [ ] Protected route middleware
- [ ] User profile page

**Week 3: tRPC Setup**
- [ ] Initialize tRPC server
- [ ] Create base routers (song, user, auth)
- [ ] Set up tRPC client in web app
- [ ] Implement input validation (Zod)
- [ ] Error handling middleware

**Week 4: Core UI**
- [ ] Home page layout
- [ ] Song listing with pagination
- [ ] Search bar (basic)
- [ ] Song card component
- [ ] Navigation & footer
- [ ] Responsive design

#### Phase 2: Core Features (Weeks 5-8)

**Week 5: Audio Player**
- [ ] AudioPlayer component
- [ ] Play/pause/seek controls
- [ ] Progress bar
- [ ] Volume control
- [ ] Audio state management (Zustand)

**Week 6: Lyrics Sync**
- [ ] LyricsPanel component
- [ ] LyricsSync class
- [ ] Highlight current line
- [ ] Seek integration
- [ ] Khmer font support

**Week 7: Karaoke Engine**
- [ ] Microphone capture (getUserMedia)
- [ ] PitchDetector (YIN algorithm)
- [ ] Real-time frequency display
- [ ] ScoreCalculator
- [ ] Transpose control

**Week 8: Scoring & Polish**
- [ ] Score display component
- [ ] Final score breakdown
- [ ] Score history (save to DB)
- [ ] PitchVisualizer (basic)
- [ ] End-of-song modal

#### Phase 3: Admin & CMS (Weeks 9-10)

**Week 9: Admin Foundation**
- [ ] Admin app routing
- [ ] Admin authentication (role-based)
- [ ] Dashboard layout
- [ ] Song listing table
- [ ] Artist/Category CRUD

**Week 10: Song Ingestion**
- [ ] Audio upload to R2
- [ ] LRC parser integration
- [ ] Song form (create/edit)
- [ ] Publish/unpublish toggle
- [ ] Basic lyrics editor

#### Phase 4: Polish & Launch (Weeks 11-12)

**Week 11: Testing & QA**
- [ ] Unit tests (lrc-parser, pitch-detector)
- [ ] Integration tests (tRPC routers)
- [ ] E2E tests (Cypress/Playwright)
- [ ] Cross-browser testing
- [ ] Mobile responsiveness
- [ ] Performance audit

**Week 12: Deployment & Launch**
- [ ] Vercel production setup
- [ ] Neon production database
- [ ] Environment variables
- [ ] CI/CD pipelines
- [ ] Monitoring (Sentry, Analytics)
- [ ] Documentation
- [ ] Launch checklist

### 10.3 Decision Gates

| Gate | Criteria | Approver |
|------|----------|----------|
| G1: Architecture | Schema finalized, tRPC contracts defined | Tech Lead |
| G2: Karaoke Core | Lyrics sync ≤50ms drift, scoring functional | Engineering |
| G3: Admin MVP | Song CRUD, upload workflow complete | Product |
| G4: Launch Ready | All P0 features complete, tests passing | Stakeholders |

---

## 11. Non-functional Requirements

### 11.1 Performance Requirements

| Metric | Requirement | Measurement |
|--------|-------------|-------------|
| First Contentful Paint | < 1.5s | Lighthouse |
| Largest Contentful Paint | < 2.5s | Lighthouse |
| Time to Interactive | < 3.5s | Lighthouse |
| API response time (p50) | < 100ms | APM |
| API response time (p99) | < 500ms | APM |
| Audio load time | < 2s | Custom metric |
| Lyrics sync drift | < 50ms | Test harness |

### 11.2 Security Requirements

| Requirement | Implementation |
|-------------|----------------|
| Authentication | NextAuth v5 with JWT |
| Authorization | Role-based (user, admin) |
| Input validation | Zod on all tRPC inputs |
| XSS prevention | React auto-escaping, CSP headers |
| CSRF protection | NextAuth built-in |
| Rate limiting | 100 req/min per IP on API |
| File upload validation | Type check, size limit, virus scan |
| HTTPS | Enforced via Vercel |
| Secrets management | Environment variables, no hardcoding |

### 11.3 Accessibility Requirements

| Requirement | Implementation |
|-------------|----------------|
| WCAG 2.1 Level AA | Target compliance |
| Keyboard navigation | All interactive elements |
| Screen reader support | Semantic HTML, ARIA labels |
| Color contrast | 4.5:1 minimum |
| Focus indicators | Visible focus rings |
| Alt text | All images |

### 11.4 Monitoring & Observability

| Layer | Tool | Purpose |
|-------|------|---------|
| Error tracking | Sentry | Frontend + backend errors |
| Analytics | Vercel Analytics | Page views, Web Vitals |
| Logging | Structured JSON to stdout | Debugging, audit trail |
| Uptime | Vercel (built-in) | Service availability |
| APM | Vercel (built-in) | Function performance |

### 11.5 Disaster Recovery

| Scenario | Mitigation |
|----------|------------|
| Database failure | Neon point-in-time recovery |
| Deployment failure | Vercel instant rollback |
| Audio storage failure | R2 redundancy (automatic) |
| Region outage | Vercel Edge distribution |

---

## 12. GitHub Spec Kit Integration

### 12.1 Repository Structure for Spec Kit

```
decisions/           # D###_Name.md - Decision logs
features/            # F###_Name.md - Feature specifications
proposals/           # P###_Name.md - Experimental designs
implementation/      # I###_Name.md - Implementation notes
```

### 12.2 Decision Log Template

```markdown
# D###: Decision Title

## Status
- [ ] Proposed
- [x] Approved
- [ ] Superseded by D###

## Context
<!-- What is the issue we're addressing? -->

## Alternatives Considered

### Option A: [Name]
**Pros:**
-

**Cons:**
-

### Option B: [Name]
**Pros:**
-

**Cons:**
-

## Decision
<!-- What is the change we're proposing? -->

## Rationale
<!-- Why is this the best choice? -->

## Impact
<!-- What are the consequences? -->

## Related
- Features: F###
- Proposals: P###
- Implementation: I###
```

### 12.3 Feature Specification Template

```markdown
# F###: Feature Name

## Problem Statement
<!-- What problem does this solve? -->

## Solution
<!-- How do we solve it? -->

## User Stories

### US1: [Story Title]
As a [type of user], I want [goal] so that [reason].

**Acceptance Criteria:**
- [ ] Criteria 1
- [ ] Criteria 2

## Implementation

### Components
- Component A: Description
- Component B: Description

### API Endpoints
- `router.procedure`: Description

### Database Changes
- Table: Changes

## Metrics

### Success Metrics
- Metric 1: Target
- Metric 2: Target

### Monitoring
- Alert: Condition

## Related
- Decisions: D###
- Implementation: I###
```

### 12.4 PR Template Integration

PRs must reference:
1. Decision log (if architectural change)
2. Feature spec (if feature work)
3. Implementation notes (if complex)

Example PR title:
```
feat(karaoke-engine): Implement YIN pitch detection [F001, D008]
```

### 12.5 Git LFS Configuration

**`.gitattributes`**:
```
# Audio files
*.mp3 filter=lfs diff=lfs merge=lfs -text
*.wav filter=lfs diff=lfs merge=lfs -text
*.ogg filter=lfs diff=lfs merge=lfs -text

# Large assets
*.zip filter=lfs diff=lfs merge=lfs -text
```

### 12.6 Release Tagging Strategy

```
v1.0.0-alpha.1    # First alpha release
v1.0.0-beta.1     # Beta testing
v1.0.0-rc.1       # Release candidate
v1.0.0            # Production release
```

Tags should reference Spec Kit docs in release notes:
```markdown
## v1.0.0

### Features
- F001: Karaoke Mode with real-time scoring
- F002: Song CMS for content management

### Decisions Implemented
- D001: Client-side karaoke engine
- D002: PostgreSQL with Neon

### Breaking Changes
None
```

---

## Appendix A: Technology Stack Summary

| Layer | Technology | Version |
|-------|------------|---------|
| Frontend | Next.js | 16.x |
| React | React | 19.x |
| Styling | Tailwind CSS | 4.x |
| UI Components | shadcn/ui | Latest |
| State Management | Zustand | 4.x |
| API | tRPC | 10.x |
| Validation | Zod | 3.x |
| Database | PostgreSQL (Neon) | 15.x |
| ORM | Prisma | 5.x |
| Authentication | NextAuth | 5.x |
| Monorepo | Turborepo | 1.x |
| Package Manager | pnpm | 8.x |
| Deployment | Vercel | - |
| Storage | Cloudflare R2 | - |
| Monitoring | Sentry | Latest |

---

## Appendix B: Environment Variables

```bash
# Database
DATABASE_URL="postgresql://..."

# Auth
NEXTAUTH_URL="https://karaoke.example.com"
NEXTAUTH_SECRET="..."
GOOGLE_CLIENT_ID="..."
GOOGLE_CLIENT_SECRET="..."

# Storage
CLOUDFLARE_ACCOUNT_ID="..."
CLOUDFLARE_R2_ACCESS_KEY_ID="..."
CLOUDFLARE_R2_SECRET_ACCESS_KEY="..."
CLOUDFLARE_R2_BUCKET_NAME="karaoke-audio"

# Monitoring
SENTRY_DSN="..."
NEXT_PUBLIC_SENTRY_DSN="..."

# Feature Flags (optional)
ENABLE_RECORDING=false
```

---

## Appendix C: Glossary

| Term | Definition |
|------|------------|
| LRC | Lyric file format with timestamps |
| YIN | Pitch detection algorithm |
| RAF | requestAnimationFrame |
| RSC | React Server Components |
| tRPC | TypeScript RPC framework |
| Neon | Serverless PostgreSQL |
| R2 | Cloudflare object storage |
| Spec Kit | Documentation framework for decisions |

---

**Document Version History:**

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 1.0.0 | 2024-12-02 | Engineering | Initial comprehensive plan |

---

*End of Document*
