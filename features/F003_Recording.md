# F003: Recording & Playback

## Problem Statement

Users want to record their karaoke sessions to review later, share with friends, or compare against previous attempts. Without recording capability, the karaoke experience is ephemeral and users miss the opportunity to track improvement.

## Solution

Add optional recording feature that:
- Records user's microphone input during karaoke session
- Optionally mixes with backing track for playback
- Allows saving recordings to user's profile
- Enables playback of past recordings

## User Stories

### US1: Record Session
As a user, I want to record my singing session so that I can listen back later.

**Acceptance Criteria:**
- [ ] "Record" toggle visible during karaoke mode
- [ ] Clear indicator when recording is active
- [ ] Recording continues throughout song
- [ ] Recording stops when song ends or user stops

### US2: Preview Recording
As a user, I want to preview my recording before saving so that I can decide if it's worth keeping.

**Acceptance Criteria:**
- [ ] Play/pause recording after session ends
- [ ] Progress bar for playback position
- [ ] Option to discard without saving
- [ ] Recording automatically discarded if not explicitly saved

### US3: Save Recording
As a user, I want to save my recording so that I can access it later.

**Acceptance Criteria:**
- [ ] "Save Recording" button after preview
- [ ] Recording uploaded to storage
- [ ] Recording linked to user and song
- [ ] Confirmation message on successful save
- [ ] Requires authentication

### US4: View Past Recordings
As a user, I want to view and play my past recordings so that I can track my progress.

**Acceptance Criteria:**
- [ ] List of recordings on profile page
- [ ] Grouped by song
- [ ] Shows date and score for each recording
- [ ] Play button for each recording
- [ ] Delete option for unwanted recordings

## Implementation

### Recording Engine

```typescript
// packages/karaoke-engine/src/audio-recorder.ts
export class AudioRecorder {
  private mediaRecorder: MediaRecorder | null = null
  private chunks: Blob[] = []

  async start(stream: MediaStream): Promise<void> {
    this.chunks = []
    this.mediaRecorder = new MediaRecorder(stream, {
      mimeType: 'audio/webm;codecs=opus'
    })

    this.mediaRecorder.ondataavailable = (e) => {
      if (e.data.size > 0) {
        this.chunks.push(e.data)
      }
    }

    this.mediaRecorder.start(1000) // Chunk every 1s
  }

  stop(): Blob {
    this.mediaRecorder?.stop()
    return new Blob(this.chunks, { type: 'audio/webm' })
  }

  getBlob(): Blob {
    return new Blob(this.chunks, { type: 'audio/webm' })
  }
}
```

### Storage Strategy

Recordings stored in Cloudflare R2:
- Path: `recordings/{userId}/{songId}/{timestamp}.webm`
- Retention: 30 days for free tier (future), unlimited for premium
- Max size: 20MB per recording

### API Endpoints

```typescript
recordingRouter
├── getUploadUrl  // Get presigned R2 URL
├── save          // Save recording metadata to DB
├── list          // User's recordings
├── get           // Single recording details
└── delete        // Delete recording
```

### Database Model

```prisma
model Recording {
  id          String   @id @default(uuid())
  userId      String
  user        User     @relation(...)
  songId      String
  song        Song     @relation(...)
  scoreId     String?  // Link to associated score
  score       Score?   @relation(...)
  recordingUrl String
  durationSec Int
  createdAt   DateTime @default(now())

  @@index([userId])
  @@index([songId])
}
```

## Priority

**P1 - Post-MVP Enhancement**

Recording is valuable but not critical for MVP launch. Core karaoke experience (F001) must work first.

## Metrics

### Success Metrics

| Metric | Target | Measurement |
|--------|--------|-------------|
| Recording feature usage | > 20% of sessions | Analytics |
| Recording save rate | > 50% of recordings | Analytics |
| Playback completion | > 70% | Analytics |

### Storage Metrics

- Average recording size
- Total storage per user
- Storage costs tracking

## Related

- Features: F001_KaraokeMode, F004_Scoring
- Decisions: D001_KaraokeEngine, D006_AudioStorage
- Implementation: I004_KaraokeEngine
