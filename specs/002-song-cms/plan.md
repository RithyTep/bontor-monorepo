# Implementation Plan: Song Content Management System

**Branch**: `002-song-cms`
**Created**: 2024-12-02
**Spec**: `specs/002-song-cms/spec.md`

---

## Summary

Implement the admin interface and backend APIs for managing songs, artists, and categories. Files are uploaded directly to Cloudflare R2 using presigned URLs, with metadata stored in PostgreSQL.

---

## Technical Context

| Aspect | Decision |
|--------|----------|
| Admin App | Next.js 16 (separate from user app) |
| File Storage | Cloudflare R2 |
| Upload Method | Presigned URLs (client → R2 direct) |
| LRC Parsing | Server-side in tRPC mutation |
| Authentication | NextAuth v5 with admin role check |
| UI Components | shadcn/ui |

---

## Constitution Check

### Phase -1: Pre-Implementation Gates

#### Simplicity Gate (Article V)
- [x] Using existing `apps/admin` application
- [x] Using existing `@karaoke/trpc` package for API
- [x] LRC parser in `@karaoke/utils` package

#### Anti-Abstraction Gate (Article VI)
- [x] Using @aws-sdk/client-s3 directly for R2
- [x] Standard tRPC patterns for admin procedures
- [x] react-dropzone for file uploads (no custom wrapper)

#### Integration-First Gate (Article VII)
- [x] Contract tests for admin.song procedures
- [x] R2 upload tested with real bucket
- [ ] LRC parser tested with real LRC files

---

## Project Structure

### Specification Directory
```
specs/002-song-cms/
├── spec.md
├── plan.md
├── data-model.md
├── contracts/
│   ├── song-admin-api.md
│   └── upload-api.md
├── quickstart.md
└── tasks.md
```

### Source Code Structure
```
packages/utils/
├── src/
│   ├── lrc-parser.ts         # LRC to JSON parser
│   └── slug-generator.ts     # URL-safe slug generation
└── tests/
    └── lrc-parser.test.ts

packages/trpc/
├── routers/
│   └── admin/
│       ├── song.ts           # Admin song CRUD
│       ├── artist.ts         # Artist CRUD
│       ├── category.ts       # Category CRUD
│       └── upload.ts         # Presigned URL generation
└── tests/
    └── admin/
        └── song.test.ts

apps/admin/
├── app/
│   ├── layout.tsx            # Admin layout with sidebar
│   ├── page.tsx              # Dashboard
│   ├── songs/
│   │   ├── page.tsx          # Song list
│   │   ├── new/page.tsx      # Create song form
│   │   └── [id]/
│   │       ├── page.tsx      # Song detail/preview
│   │       └── edit/page.tsx # Edit song
│   ├── artists/
│   │   ├── page.tsx
│   │   └── new/page.tsx
│   └── categories/
│       └── page.tsx
├── components/
│   ├── forms/
│   │   ├── SongForm.tsx
│   │   ├── ArtistForm.tsx
│   │   └── CategoryForm.tsx
│   ├── uploaders/
│   │   ├── AudioUploader.tsx
│   │   ├── LrcUploader.tsx
│   │   └── ImageUploader.tsx
│   └── layout/
│       ├── AdminSidebar.tsx
│       └── AdminHeader.tsx
└── lib/
    └── admin-trpc.ts
```

---

## Technical Design

### File Upload Flow

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        FILE UPLOAD FLOW                                  │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  1. Admin selects files in browser                                       │
│         │                                                                │
│         ▼                                                                │
│  2. Client requests presigned URLs                                       │
│     POST /api/trpc/admin.upload.getUrls                                 │
│     { songId, hasAudio: true, hasLrc: true, hasCover: false }          │
│         │                                                                │
│         ▼                                                                │
│  3. Server generates presigned PUT URLs (valid 1 hour)                  │
│     Response: { audioUrl, lrcUrl }                                      │
│         │                                                                │
│         ▼                                                                │
│  4. Client uploads directly to R2                                        │
│     PUT audioUrl → audio.mp3                                            │
│     PUT lrcUrl → lyrics.lrc                                             │
│     (with progress tracking via XHR)                                    │
│         │                                                                │
│         ▼                                                                │
│  5. Client confirms upload complete                                      │
│     POST /api/trpc/admin.song.create                                    │
│     { title, artistId, ..., lrcContent (for parsing) }                 │
│         │                                                                │
│         ▼                                                                │
│  6. Server parses LRC, extracts duration, saves to DB                   │
│                                                                          │
└──────────────────────────────────────────────────────────────────────────┘
```

### R2 Bucket Structure

```
karaoke-assets/
├── audio/
│   └── {songId}/
│       └── audio.mp3
├── lyrics/
│   └── {songId}/
│       └── lyrics.lrc
├── covers/
│   └── {songId}/
│       └── cover.jpg
└── artists/
    └── {artistId}/
        └── image.jpg
```

### LRC Parser

```typescript
// packages/utils/src/lrc-parser.ts

interface LyricsLine {
  t: number  // timestamp in seconds
  l: string  // lyric text
}

interface ParseResult {
  lyrics: LyricsLine[]
  metadata: Record<string, string>
  errors: string[]
}

function parseLrc(content: string): ParseResult {
  // Parse [mm:ss.xx] timestamps
  // Extract metadata tags [ti:], [ar:], etc.
  // Sort by timestamp
  // Return structured data
}
```

---

## File Creation Order

### Phase 1: Utils Package

1. `packages/utils/tests/lrc-parser.test.ts`
2. `packages/utils/src/lrc-parser.ts`
3. `packages/utils/src/slug-generator.ts`

### Phase 2: Admin API Routes

4. `packages/trpc/routers/admin/upload.ts` - Presigned URL generation
5. `packages/trpc/routers/admin/song.ts` - Song CRUD
6. `packages/trpc/routers/admin/artist.ts` - Artist CRUD
7. `packages/trpc/routers/admin/category.ts` - Category CRUD
8. `packages/trpc/tests/admin/song.test.ts`

### Phase 3: Admin UI Components

9. `apps/admin/components/uploaders/AudioUploader.tsx`
10. `apps/admin/components/uploaders/LrcUploader.tsx`
11. `apps/admin/components/uploaders/ImageUploader.tsx`
12. `apps/admin/components/forms/SongForm.tsx`
13. `apps/admin/components/forms/ArtistForm.tsx`
14. `apps/admin/components/forms/CategoryForm.tsx`

### Phase 4: Admin Pages

15. `apps/admin/app/layout.tsx` - Admin layout with auth guard
16. `apps/admin/app/songs/page.tsx` - Song list with filters
17. `apps/admin/app/songs/new/page.tsx` - Create song
18. `apps/admin/app/songs/[id]/page.tsx` - Song detail/preview
19. `apps/admin/app/songs/[id]/edit/page.tsx` - Edit song
20. `apps/admin/app/artists/page.tsx`
21. `apps/admin/app/categories/page.tsx`

---

## Database Schema Updates

```prisma
model Song {
  id          String   @id @default(uuid())
  title       String
  slug        String   @unique
  artistId    String
  categoryId  String
  audioUrl    String   // R2 public URL
  lyricsJson  Json     // Parsed LRC as JSON
  coverUrl    String?
  durationSec Int      // Extracted from audio/LRC
  language    String?  // 'km', 'en', etc.
  isPublished Boolean  @default(false)
  publishedAt DateTime?
  playCount   Int      @default(0)
  createdAt   DateTime @default(now())
  updatedAt   DateTime @updatedAt

  artist   Artist   @relation(fields: [artistId], references: [id])
  category Category @relation(fields: [categoryId], references: [id])

  @@index([artistId])
  @@index([categoryId])
  @@index([isPublished, createdAt(sort: Desc)])
}

model Artist {
  id        String   @id @default(uuid())
  name      String
  slug      String   @unique
  bio       String?
  imageUrl  String?
  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt

  songs Song[]
}

model Category {
  id           String   @id @default(uuid())
  name         String
  slug         String   @unique
  displayOrder Int      @default(0)
  createdAt    DateTime @default(now())
  updatedAt    DateTime @updatedAt

  songs Song[]

  @@index([displayOrder])
}

model AdminAuditLog {
  id         String   @id @default(uuid())
  adminId    String
  action     String   // 'song.create', 'song.publish', etc.
  targetType String   // 'song', 'artist', 'category'
  targetId   String
  details    Json?
  createdAt  DateTime @default(now())

  admin User @relation(fields: [adminId], references: [id])

  @@index([adminId])
  @@index([targetType, targetId])
}
```

---

## Complexity Tracking

| Article | Deviation | Justification | Date |
|---------|-----------|---------------|------|
| None | N/A | N/A | - |

---

## Risk Mitigation

| Risk | Mitigation |
|------|------------|
| Large file upload timeout | Presigned URLs valid for 1 hour; chunked upload for >50MB |
| LRC encoding issues | Detect encoding, convert to UTF-8 |
| R2 access errors | Retry with exponential backoff |
| Admin session expires during upload | Keep session alive, handle gracefully |
