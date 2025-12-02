# F002: Song CMS

## Problem Statement

Content administrators need an efficient way to manage the song catalog: uploading audio files, creating lyrics with timestamps, organizing by artist and category, and controlling publication status. Without a proper CMS, content operations become error-prone and time-consuming.

## Solution

Build an admin CMS that allows content team to:
- Upload and manage audio files (MP3)
- Upload and parse LRC lyrics files
- Create and edit song metadata
- Manage artists and categories
- Control song publication status

## User Stories

### US1: Upload New Song
As an admin, I want to upload a new song with audio and lyrics so that users can sing it.

**Acceptance Criteria:**
- [ ] Form accepts song title, artist, category
- [ ] Audio upload supports MP3 format (max 50MB)
- [ ] LRC file upload with automatic parsing to JSON
- [ ] Duration auto-detected from audio file
- [ ] Preview audio before saving
- [ ] Song saved as draft (unpublished) initially

### US2: Edit Existing Song
As an admin, I want to edit song details so that I can fix errors or update content.

**Acceptance Criteria:**
- [ ] All fields editable (title, artist, category, audio, lyrics)
- [ ] Replacing audio updates duration automatically
- [ ] Replacing LRC re-parses timestamps
- [ ] History of edits visible
- [ ] Publish/unpublish toggle

### US3: Manage Artists
As an admin, I want to create and manage artists so that songs are properly attributed.

**Acceptance Criteria:**
- [ ] Create artist with name, image, bio
- [ ] Edit existing artists
- [ ] View songs by artist
- [ ] Cannot delete artist with songs

### US4: Manage Categories
As an admin, I want to create and manage categories so that users can filter songs.

**Acceptance Criteria:**
- [ ] Create category with name
- [ ] Edit existing categories
- [ ] View songs by category
- [ ] Cannot delete category with songs

### US5: Search and Filter Songs
As an admin, I want to search and filter the song list so that I can find specific songs quickly.

**Acceptance Criteria:**
- [ ] Search by title, artist name
- [ ] Filter by category
- [ ] Filter by publish status (draft/published)
- [ ] Sort by date created, title, artist
- [ ] Pagination for large catalogs

## Implementation

### Admin App Structure

```
apps/admin/
├── app/
│   ├── dashboard/
│   │   └── page.tsx          # Stats overview
│   ├── songs/
│   │   ├── page.tsx          # Song listing
│   │   ├── new/
│   │   │   └── page.tsx      # Create song
│   │   └── [id]/
│   │       └── edit/
│   │           └── page.tsx  # Edit song
│   ├── artists/
│   │   ├── page.tsx          # Artist listing
│   │   ├── new/
│   │   └── [id]/edit/
│   ├── categories/
│   │   └── page.tsx          # Category listing
│   └── layout.tsx            # Admin layout with sidebar
└── components/
    ├── SongForm.tsx
    ├── AudioUploader.tsx
    ├── LyricsEditor.tsx
    ├── SongTable.tsx
    └── ArtistForm.tsx
```

### API Endpoints (Admin tRPC)

```typescript
adminRouter
├── song
│   ├── list       // with filters
│   ├── get        // by ID
│   ├── create     // with audio + LRC
│   ├── update     // partial update
│   ├── delete     // soft delete
│   └── publish    // toggle publish status
├── artist
│   ├── list
│   ├── create
│   ├── update
│   └── delete
└── category
    ├── list
    ├── create
    └── delete
```

### File Upload Flow

```
1. Admin selects files (MP3 + LRC)
   ↓
2. Client validates file types and sizes
   ↓
3. Upload to Cloudflare R2 via presigned URL
   ↓
4. Backend parses LRC → JSON
   ↓
5. Backend extracts audio duration
   ↓
6. Save song record to database
```

### LRC Parser

```typescript
// packages/utils/lrc-parser.ts
export function parseLrc(lrc: string): LyricsLine[] {
  const lines = lrc.split(/\r?\n/)
  const result: LyricsLine[] = []
  const timeRegex = /\[(\d+):(\d+)(?:\.(\d+))?\]/g

  for (const line of lines) {
    // Parse timestamps and text
    // Handle multiple timestamps per line
    // Sort by time
  }

  return result.sort((a, b) => a.t - b.t)
}
```

## Metrics

### Success Metrics

| Metric | Target | Measurement |
|--------|--------|-------------|
| Song upload success rate | > 95% | Admin analytics |
| Time to upload song | < 3 min | User testing |
| LRC parse errors | < 5% | Error logs |
| Admin satisfaction | > 4/5 | Survey |

### Monitoring

- Upload failures tracked in Sentry
- R2 storage usage monitoring
- Parse error logging for LRC issues

## Related

- Features: F001_KaraokeMode, F005_AdminPanel
- Decisions: D002_DBSchema, D006_AudioStorage, D011_LyricsFormat
- Implementation: I003_CMSWorkflow
