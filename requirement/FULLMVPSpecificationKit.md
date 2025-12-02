Nice — here's the **FULL MVP Specification Kit (SpecKit)** for the **Karaoke Cambodia** product, written as a detailed, engineer-friendly spec you can hand to product managers, designers, backend and frontend engineers, QA, and DevOps. It ties directly to the monorepo scaffold you asked for (Next.js 16 + Vercel + Neon + Prisma + tRPC + Sanity).

I’ll cover: goal, target users, success metrics, full feature list, acceptance criteria, user flows & wireframe notes, API & tRPC contracts, DB / Prisma schema, Sanity CMS schema, file formats (LRC → JSON), karaoke engine design (lyrics sync + scoring + pitch detection), frontend components & pages, admin interfaces, infra & deployment, CI/CD checklist, testing & QA matrix, security/privacy, monitoring, and an itemized deliverables checklist.

---

# MVP SpecKit — Karaoke Cambodia (v1.0)

## 1 — Goal & Vision

**Goal:** Deliver a simple, reliable, production-ready web karaoke experience optimized for Cambodia: fast song discovery, accurate Khmer lyrics sync, local audio hosting, basic pitch scoring, user favorites, and a CMS for admins to add/manage songs.

**Non-goal for MVP:** social/live multiplayer rooms, mobile native apps, advanced ML-based scoring, offline PWA, paid subscriptions.

---

## 2 — Target Users

* Casual home singers in Cambodia (Khmer + English)
* Bar/karaoke host staff (admin uploads songs)
* Product admin/content team (managing metadata & media)
* QA & support teams

---

## 3 — Success Metrics (KPIs)

* Song page load time ≤ 1s (cold).
* Search latency ≤ 200 ms for common queries.
* Lyrics sync drift ≤ 50 ms after initial sync.
* At least 80% match-rate between human-labeled pitch correctness and algorithm score on test set.
* 7-day retention > X% (project KPI, set by PM).
* Errors < 0.5% of API calls under normal load.

---

## 4 — MVP Feature List (detailed)

### 4.1 Public / Consumer

* Home / song listing (paginated)
* Search (Khmer + English, fuzzy)
* Category filtering (Khmer / Thai / Chinese / English)
* Song details page:

  * Audio player
  * Synced lyrics highlighting (LRC -> JSON)
  * "Start Singing" / Karaoke mode (mic)
  * Pitch visualizer + final score (0–100)
  * Add to favorites
* User auth (Google + email)
* Profile page with favorites list

### 4.2 Admin / CMS

* Sanity-based CMS or Sanity-like admin:

  * CRUD songs (title, artist, category)
  * Upload MP3 & LRC files
  * Publish/unpublish
  * Artist CRUD
  * Category CRUD
* Admin dashboard: list, search, publish control

### 4.3 Platform & Infra

* Next.js 16 frontend on Vercel
* Backend tRPC server (on Vercel Serverless or Fastify on Vercel)
* Neon Postgres + Prisma
* Storage: Vercel Blob or Cloudflare R2 for audio & LRC
* Simple search using PostgreSQL pg_trgm or Meilisearch (option)
* CI pipeline + automated migrations + tests

---

## 5 — Acceptance Criteria (examples)

**Song details page**

* GIVEN a published song
* WHEN user visits /song/[id]
* THEN page shows title, artist, category, play button, and lyrics panel and audio loads within 1s

**Lyrics sync**

* Given an LRC with timestamps
* When audio plays normal speed
* Then highlighted lyric line matches LRC timestamp within ±50ms under test harness

**Score**

* Given prerecorded reference melody & user mic recording
* Then system computes a numeric score between 0–100 and displays breakdown: pitchAccuracy %, durationCovered %

---

## 6 — User Flows & Wireframe Notes

### Primary flow: search → play → sing → score → favorite

1. Home: top search bar + categories
2. Search results: show song card (play preview, duration)
3. Song page: full player + lyrics pane
4. Karaoke Mode:

   * Request mic permission
   * Show waveform + pitch dots vs reference pitch line
   * Real-time feedback: green dot for in-tune, red for off
   * End: show numeric score + share/save option (share later)

Wireframe callouts:

* Lyrics pane should support Khmer fonts (Unicode, font fallback)
* Big “Start Singing” CTA pinned at bottom in mobile
* Minimal latency focus for audio UI (avoid heavy re-renders)

---

## 7 — Data Model & Prisma Schema (complete)

Below is the Prisma schema for the MVP. Put this at `packages/db/prisma/schema.prisma`.

```prisma
generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

model User {
  id        String    @id @default(uuid())
  email     String    @unique
  name      String?
  image     String?
  favorites Favorite[]
  createdAt DateTime  @default(now())
}

model Artist {
  id    String @id @default(uuid())
  name  String
  songs Song[]
}

model Category {
  id    String @id @default(uuid())
  name  String
  slug  String  @unique
  songs Song[]
}

model Song {
  id          String   @id @default(uuid())
  title       String
  artistId    String
  artist      Artist   @relation(fields: [artistId], references: [id])
  categoryId  String
  category    Category @relation(fields: [categoryId], references: [id])
  audioUrl    String
  lrcUrl      String?   // original LRC file (optional)
  lyricsJson  Json      // parsed LRC -> [{t:number,l:string}]
  durationSec Int
  language    String?   // "km", "en", etc.
  isPublished Boolean   @default(false)
  createdAt   DateTime  @default(now())
  updatedAt   DateTime  @updatedAt
}

model Favorite {
  id        String   @id @default(uuid())
  userId    String
  user      User     @relation(fields: [userId], references: [id])
  songId    String
  song      Song     @relation(fields: [songId], references: [id])
  createdAt DateTime @default(now())
}
```

Notes:

* `lyricsJson` stores parsed timestamps for front-end fast access.
* Use JSONB column type via Prisma `Json`.

---

## 8 — Sanity CMS Schema (example)

Place these schemas in the Sanity studio:

`song.js`

```js
export default {
  name: 'song',
  title: 'Song',
  type: 'document',
  fields: [
    { name: 'title', type: 'string', title: 'Title' },
    { name: 'artist', type: 'reference', to: [{type:'artist'}] },
    { name: 'category', type: 'reference', to: [{type:'category'}] },
    { name: 'audioFile', type: 'file', title: 'Audio (MP3)' },
    { name: 'lrcFile', type: 'file', title: 'LRC file' },
    { name: 'lyricsJson', type: 'array', of:[{type:'object', fields:[{name:'t',type:'number'},{name:'l',type:'string'}]}] },
    { name: 'durationSec', type: 'number' },
    { name: 'isPublished', type: 'boolean', initialValue: false }
  ]
}
```

`artist.js`, `category.js` are simple documents with `name` and `image`.

Sanity upload webhooks:

* On publish, trigger a webhook to backend API to ingest and parse LRC and add record to Neon DB.

---

## 9 — File formats & converters

### LRC format → JSON

* LRC line: `[mm:ss.xx] lyric text`
* Convert to JSON: `[{"t": 12.435, "l": "ខ្ញុំស្រលាញ់អ្នក"}]`
* Parser should:

  * Support multiple timestamps per line
  * Ignore metadata `[ti:, ar:, al:]`
  * Normalize milliseconds to 3 digits

Example parser behavior available in scaffold `packages/utils/lrc-parser.ts`.

---

## 10 — tRPC API Contracts (recommended)

We use tRPC for type-safe FE↔BE. Example routers & types.

### Router: `song.list`

Input:

```ts
{ q?: string, category?: string, take?: number, skip?: number }
```

Output:

```ts
{ items: SongDTO[], total: number }
```

### Router: `song.get`

Input: `{ id: string }`
Output: `SongDTO` (see packages/types)

### Router: `favorite.add`

Input: `{ songId: string }`
Output: `{ ok: true }`

### Router: `score.evaluate` (optional server-side scoring)

Input:

```ts
{ songId: string, recordingUrl: string } // server-side compare
```

Output:

```ts
{ score: number, details: { pitchAccuracy: number, durationCoverage: number } }
```

tRPC types are defined in `packages/trpc` and `packages/types`.

---

## 11 — Karaoke Engine Design (frontend-heavy)

Two parts: **Lyrics Sync** and **Scoring/Pitch Detection**.

### 11.1 Lyrics Sync

* Use Web Audio API to get current playback `currentTime`.
* Keep an indexed pointer to the next lyric timestamp.
* Use `requestAnimationFrame` to update highlight; avoid interval timers to minimize drift.
* On seek, update pointer by binary searching `lyricsJson` by time.
* For small songs, store `lyricsJson` on page load. For huge lyrics, fetch streaming.

Pseudocode:

```ts
let idx = 0;
function raf() {
  const t = audio.currentTime;
  while (idx + 1 < lyrics.length && lyrics[idx+1].t <= t) idx++;
  renderHighlight(idx);
  requestAnimationFrame(raf);
}
```

### 11.2 Pitch Detection (MVP)

* Use Web Audio API `getUserMedia` to capture mic.
* Route mic stream to `AnalyserNode` and `ScriptProcessor` or `AudioWorklet`.
* Implement **YIN** or **auto-correlation** algorithm in JS for pitch detection (YIN recommended for stability).
* Convert pitch to MIDI/Hz; compare to reference pitch per timestamp.
* Scoring:

  * Frame-based: every analyzed audio frame (e.g., 50ms) compute whether frequency is within tolerance of reference pitch for that timestamp.
  * Score = weighted ratio: `score = 100 * (sum(correctFramesWeight) / sum(totalFramesWeight))`.
  * Add penalties for long silence or huge deviation.

MVP simplification:

* Precompute reference pitch line from the song’s melody (either manual or generated via offline tool).
* For initial MVP, simplify to pitch contour matching rather than note-onset alignment.

Implementation notes:

* Use `AudioWorklet` if supported to reduce jitter.
* Allow `transpose` option to match user's vocal range.

---

## 12 — Frontend Components & Pages (mapping to scaffold)

Top-level pages:

* `/` — Home (featured + search bar)
* `/songs` — Song listing (paginated)
* `/song/[id]` — Song details + karaoke mode
* `/auth` — Login flow
* `/profile` — Favorites & user info
* `/admin/*` — Admin hooks (optional; main in Sanity)

Key components:

* `AudioPlayer` (controls, progress, speed)
* `LyricsPanel` (highlight lines)
* `KaraokeMode` (mic permission, pitch visualizer, score)
* `SearchBar` (typeahead)
* `SongCard`
* Shared UI: `Button`, `Modal`, `Toast`

State:

* Global: user session, playback state, currentSong
* Local: lyrics pointer, current score components

Performance:

* Use RSC + client components only where necessary.
* Memoize heavy visualizer updates.
* Avoid re-rendering whole lyrics pane each animation frame — update DOM element class for active line instead.

---

## 13 — Admin & CMS Workflow

* Content team uses Sanity Studio to upload songs & LRC.
* On publish, Sanity webhook posts to backend `/api/ingest-song`:

  * Backend downloads audio and LRC to blob storage
  * Parses LRC → JSON and stores JSON in DB (or stores JSON in object store and URL in DB)
  * Creates/updates Song row in Neon DB via Prisma
* Admin can search & correct timestamp edits via Sanity or a custom admin UI.

---

## 14 — API & Infra Spec

* Host front-end on Vercel (Edge + App Router).
* Host tRPC API on Vercel Serverless (or as a Fastify container if you prefer).
* Database: Neon Postgres. Use connection pooling best practices (serverless-friendly).
* Storage: Vercel Blob (simple) or Cloudflare R2 (cost-effective for video/audio).
* Domain: `karaoke.example.kh` with TLS via Vercel.

Secrets:

* `DATABASE_URL`, `SANITY_TOKEN`, `CLOUDFLARE_R2_*`, `NEXTAUTH_SECRET`, `GOOGLE_CLIENT_ID/SECRET`

Backups & migrations:

* Use Prisma migrations via `prisma migrate deploy` in CI.
* Regular DB backups (Neon handles snapshots). Export critical audio metadata regularly.

---

## 15 — CI/CD Checklist (suggested)

**CI (on push & PR):**

* `pnpm install`
* `pnpm -w lint` (ESLint + Prettier)
* `pnpm -w test` (unit tests + typecheck)
* `pnpm -w build` (turbo build)
* `pnpm -w prisma generate` (ensure client builds)
* Run a light integration test (song page render smoke test)

**CD (on merge to main/master):**

* Deploy web to Vercel (automatic)
* Run DB migrations via `prisma migrate deploy` on deploy job (careful with serverless)
* Sanity auto-deploy (if needed)
* Run smoke tests (post-deploy)

Rollback:

* Vercel automatic rollbacks via prior deployment
* DB migrations: use incremental, reversible migrations and backups

---

## 16 — Testing & QA Plan

**Unit tests**:

* lrc parser edge cases
* pitch detection math
* tRPC routers input validation (zod)

**Integration tests**:

* API endpoints with test DB (seeded)
* FE rendering of song page with mocked tRPC client

**E2E tests**:

* Cypress: search → open song → play audio (mocked) → lyrics highlight works
* Simulated mic input via pre-recorded audio (for scoring validation)

**Performance tests**:

* Load test API endpoints with k6 or locust (search & song fetch)
* Cold-start and warm-start latency for serverless functions

**QA matrix**:

* Functional test checklist per feature (listed as acceptance tests earlier)
* Cross-browser: Chrome, Firefox, Safari (desktop & mobile viewport)
* Khmer fonts rendering tests across platforms

---

## 17 — Security & Privacy

* Auth: NextAuth with JWT or session cookies (HTTPS only)
* Rate-limit endpoints (search?, score endpoints) to prevent abuse
* Validate & sanitize all uploaded files (limit size, scan for threats)
* Store personal data minimal (email, name). Comply with local laws on user data retention.
* CORS restrictions for backend endpoints.
* Use Signed URLs for audio access if you want time-limited access.

---

## 18 — Monitoring & Observability

* Error tracking: Sentry (frontend & backend)
* Logging: structured logs to an ELK or Datadog; at minimum, Vercel logs and serverless tracing
* Metrics: Prometheus-compatible metrics via a collector, or use Vercel analytics + custom logs
* Uptime checks: Ping home and critical API endpoints

---

## 19 — Scalability & Ops Notes

* Neon scales vertically but keep queries efficient (indexes: title, artistId, categoryId; pg_trgm for search).
* Offload heavy CPU tasks (server-side scoring or preprocessing) to worker functions or containers (e.g., Cloud Run or Fargate).
* For many concurrent mic sessions, push real-time state to Vercel Realtime or Pusher.

---

## 20 — Deliverables Checklist (what engineering gets)

**Product / PM**

* This SpecKit doc

**Frontend**

* Next.js app (pages & components listed)
* Karaoke Mode UI with pitch visualizer (MVP)
* tRPC client & integration

**Backend**

* tRPC routers: song, artist, category, favorite, auth
* Ingest endpoint for Sanity webhook
* Prisma client & migrations

**DB**

* Prisma schema above + seed scripts

**CMS**

* Sanity schema files for song/artist/category

**Infra**

* `turbo.json`, root `package.json`, `pnpm-workspace.yaml`
* Dockerfiles + deploy scripts
* CI workflow (YAML) for lint/build/test/deploy

**QA**

* Unit tests for utils (lrc parser)
* Cypress E2E basic scenario

---

## 21 — Roadmap / Next-features (post-MVP suggestions)

* Live rooms & synchronized multi-user sessions
* Leaderboards & social sharing
* Offline PWA
* Paid content & subscriptions
* Mobile native wrappers (Tauri / Capacitor / Expo)
* Auto-extract melody from audio (ML) for automatic pitch reference

---

## 22 — Appendix: Example tRPC Input/Output Types (TypeScript)

`packages/types/song.ts`

```ts
export type LyricsLine = { t: number; l: string }
export interface SongDTO {
  id: string
  title: string
  artistId: string
  artistName?: string
  categoryId: string
  audioUrl: string
  lrcUrl?: string
  lyricsJson: LyricsLine[]
  durationSec: number
  isPublished: boolean
}
```

Example router signature (song.get):

```ts
// input zod
z.object({ id: z.string().uuid() })
// returns SongDTO
```

---