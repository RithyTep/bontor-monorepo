# F004: Scoring System

## Problem Statement

Users want objective feedback on their singing performance. Without a scoring system, there's no gamification element to encourage practice and improvement. Users need quantifiable metrics to understand how well they're singing.

## Solution

Implement a real-time scoring system that:
- Compares user's pitch to reference pitch
- Calculates accuracy percentage per frame
- Provides final score (0-100) with breakdown
- Saves scores to user history for progress tracking

## User Stories

### US1: Real-time Score Updates
As a user, I want to see my score updating in real-time so that I can adjust my singing.

**Acceptance Criteria:**
- [ ] Score counter visible during karaoke mode
- [ ] Updates every few seconds (not every frame to reduce distraction)
- [ ] Visual indication when singing is on-pitch
- [ ] Score doesn't decrease (accumulates correct frames)

### US2: Final Score Summary
As a user, I want to see a detailed score breakdown at the end so that I understand my performance.

**Acceptance Criteria:**
- [ ] Modal/screen appears when song ends
- [ ] Total score prominently displayed (0-100)
- [ ] Breakdown: pitch accuracy %, duration coverage %
- [ ] Comparison to previous best (if exists)
- [ ] Option to save or dismiss

### US3: Score History
As a user, I want to view my score history for each song so that I can track improvement.

**Acceptance Criteria:**
- [ ] List of past scores on song detail page
- [ ] Shows date, score, breakdown for each attempt
- [ ] Personal best highlighted
- [ ] Chart showing improvement over time (optional, P2)

### US4: Session Settings
As a user, I want to adjust scoring sensitivity so that the game matches my skill level.

**Acceptance Criteria:**
- [ ] Tolerance slider (easier = wider tolerance)
- [ ] Settings explained in tooltip
- [ ] Score reflects adjusted difficulty
- [ ] Settings saved per session

## Implementation

### Scoring Algorithm

```typescript
// packages/karaoke-engine/src/score-calculator.ts

interface ScoreFrame {
  timestamp: number
  detectedHz: number | null
  referenceHz: number | null
  isCorrect: boolean
}

export class ScoreCalculator {
  private frames: ScoreFrame[] = []
  private toleranceCents: number

  constructor(toleranceCents: number = 80) {
    this.toleranceCents = toleranceCents
  }

  addFrame(detected: number | null, reference: number | null): void {
    const isCorrect = this.isOnPitch(detected, reference)
    this.frames.push({
      timestamp: Date.now(),
      detectedHz: detected,
      referenceHz: reference,
      isCorrect
    })
  }

  private isOnPitch(detected: number | null, reference: number | null): boolean {
    if (detected === null || reference === null) return false
    const cents = Math.abs(1200 * Math.log2(detected / reference))
    return cents <= this.toleranceCents
  }

  getScore(): number {
    const validFrames = this.frames.filter(f => f.referenceHz !== null)
    if (validFrames.length === 0) return 0
    const correct = validFrames.filter(f => f.isCorrect).length
    return Math.round((correct / validFrames.length) * 100)
  }

  getBreakdown(): ScoreBreakdown {
    const total = this.frames.length
    const withPitch = this.frames.filter(f => f.detectedHz !== null).length
    const correct = this.frames.filter(f => f.isCorrect).length

    return {
      totalScore: this.getScore(),
      pitchAccuracy: total > 0 ? correct / total : 0,
      durationCoverage: total > 0 ? withPitch / total : 0,
      correctFrames: correct,
      totalFrames: total
    }
  }
}
```

### Reference Pitch Data

For accurate scoring, songs need reference pitch data:

**MVP Approach:**
- Manually create pitch contours for launch songs
- Use simplified melody line (note per lyric line)
- Store as JSON: `[{t: number, hz: number}]`

**Future Enhancement:**
- ML-based melody extraction from audio
- Community-contributed reference data

### Database Model

```prisma
model Score {
  id              String   @id @default(uuid())
  userId          String
  user            User     @relation(...)
  songId          String
  song            Song     @relation(...)
  totalScore      Int      // 0-100
  pitchAccuracy   Float    // 0.0-1.0
  durationCoverage Float   // 0.0-1.0
  transposeSemitones Int   @default(0)
  toleranceCents  Int      @default(80)
  createdAt       DateTime @default(now())

  @@index([userId, songId])
  @@index([songId, totalScore])
}
```

### API Endpoints

```typescript
scoreRouter
├── submit    // Save score after session
├── history   // Get user's scores for a song
├── best      // Get user's best score for a song
└── leaderboard // (P2) Top scores for a song
```

## Metrics

### Success Metrics

| Metric | Target | Measurement |
|--------|--------|-------------|
| Score submission rate | > 30% of sessions | Analytics |
| Repeat play rate (same song) | > 40% | Analytics |
| Score improvement (repeat users) | Positive trend | Analytics |

### Quality Metrics

| Metric | Target | Measurement |
|--------|--------|-------------|
| Score correlation with human judgment | > 0.8 | User study |
| Score consistency (same recording) | ±3 points | Test harness |

## Related

- Features: F001_KaraokeMode, F003_Recording
- Decisions: D001_KaraokeEngine, D008_PitchDetection
- Implementation: I004_KaraokeEngine
