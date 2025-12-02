# Data Model: Karaoke Mode

**Feature**: `001-karaoke-mode`
**Created**: 2024-12-02

---

## Core Entities

### LyricsLine

Represents a single line of lyrics with its timestamp.

```typescript
interface LyricsLine {
  /** Time in seconds when this line should be highlighted */
  t: number

  /** The lyric text (Khmer or English) */
  l: string
}
```

**Constraints**:
- `t` must be ≥ 0
- `t` values must be sorted in ascending order in the array
- `l` can be empty string (instrumental section marker)

**Example**:
```json
[
  { "t": 0, "l": "" },
  { "t": 5.2, "l": "First line of the song" },
  { "t": 10.8, "l": "Second line here" },
  { "t": 15.0, "l": "បទភ្លេងខ្មែរ" }
]
```

---

### KaraokeSession

Runtime state for an active karaoke session.

```typescript
interface KaraokeSession {
  /** Unique session identifier */
  id: string

  /** Song being performed */
  songId: string

  /** Current playback state */
  isPlaying: boolean

  /** Whether microphone is active */
  isSinging: boolean

  /** Current audio position in seconds */
  currentTime: number

  /** Index of currently highlighted lyric line */
  currentLyricIndex: number

  /** Last detected pitch frequency in Hz (null if no voice) */
  detectedFrequency: number | null

  /** Transpose setting in semitones (-12 to +12) */
  transposeSemitones: number

  /** Pitch matching tolerance in cents (default: 80) */
  toleranceCents: number

  /** Count of frames where pitch was correct */
  framesCorrect: number

  /** Total frames analyzed */
  framesTotal: number

  /** Percentage of song duration where voice was detected */
  durationCoverage: number

  /** Session start timestamp */
  startedAt: Date

  /** Session end timestamp (null if ongoing) */
  endedAt: Date | null
}
```

**State Transitions**:
```
[Initial] → play → [Playing] → pause → [Paused] → play → [Playing]
                                                       → end → [Completed]
[Playing] → end → [Completed]
```

---

### ScoreFrame

Individual pitch comparison result for a single analysis frame.

```typescript
interface ScoreFrame {
  /** Timestamp in seconds when this frame was captured */
  timestamp: number

  /** Detected pitch frequency in Hz (null if no voice detected) */
  detectedFrequency: number | null

  /** Expected pitch frequency in Hz (null if no reference available) */
  expectedFrequency: number | null

  /** Difference in cents between detected and expected (-1200 to +1200) */
  centsDifference: number | null

  /** Whether this frame counts as "correct" given tolerance */
  isCorrect: boolean

  /** Confidence level of pitch detection (0-1) */
  confidence: number
}
```

**Scoring Logic**:
```typescript
function isFrameCorrect(frame: ScoreFrame, toleranceCents: number): boolean {
  // No voice detected
  if (frame.detectedFrequency === null) return false

  // No reference pitch - count as correct if voice detected
  if (frame.expectedFrequency === null) return true

  // Within tolerance (accounting for octave equivalence)
  const diff = Math.abs(frame.centsDifference ?? 0) % 1200
  return diff <= toleranceCents || diff >= (1200 - toleranceCents)
}
```

---

### FinalScore

Aggregated score after session completion.

```typescript
interface FinalScore {
  /** Session identifier */
  sessionId: string

  /** User ID (null for anonymous) */
  userId: string | null

  /** Song ID */
  songId: string

  /** Overall score (0-100) */
  totalScore: number

  /** Pitch accuracy (0-1, percentage of correct frames) */
  pitchAccuracy: number

  /** Duration coverage (0-1, percentage of song sung) */
  durationCoverage: number

  /** Settings used during session */
  settings: {
    transposeSemitones: number
    toleranceCents: number
  }

  /** Whether this is user's personal best for this song */
  isPersonalBest: boolean

  /** Timestamp of completion */
  completedAt: Date
}
```

**Score Formula**:
```typescript
function calculateFinalScore(
  pitchAccuracy: number,
  durationCoverage: number
): number {
  // Weighted average: pitch 70%, duration 30%
  const raw = (pitchAccuracy * 0.7 + durationCoverage * 0.3) * 100
  return Math.round(raw)
}
```

---

### PitchDetectionResult

Output from the YIN pitch detection algorithm.

```typescript
interface PitchDetectionResult {
  /** Detected frequency in Hz (null if no pitch detected) */
  frequency: number | null

  /** Confidence of detection (0-1, higher is more confident) */
  confidence: number

  /** Whether signal has sufficient amplitude for analysis */
  hasSignal: boolean

  /** RMS amplitude of the input signal */
  amplitude: number
}
```

---

## Database Schema (Score Persistence)

```prisma
model Score {
  id               String   @id @default(uuid())
  userId           String
  songId           String
  totalScore       Int      // 0-100
  pitchAccuracy    Float    // 0.0-1.0
  durationCoverage Float    // 0.0-1.0
  transposeSemitones Int    @default(0)
  toleranceCents   Int      @default(80)
  createdAt        DateTime @default(now())

  user User @relation(fields: [userId], references: [id])
  song Song @relation(fields: [songId], references: [id])

  @@index([userId, songId])
  @@index([songId, totalScore(sort: Desc)])
}
```

---

## Zustand Store State

```typescript
interface KaraokeStoreState {
  // Session state
  isPlaying: boolean
  isSinging: boolean
  currentTime: number
  currentLyricIndex: number

  // Pitch detection state
  detectedFrequency: number | null
  pitchHistory: number[] // Last 30 frames for visualization

  // Score state
  framesCorrect: number
  framesTotal: number
  durationCoverage: number

  // Settings
  transposeSemitones: number
  toleranceCents: number

  // Actions
  setPlaying: (playing: boolean) => void
  setSinging: (singing: boolean) => void
  updateTime: (time: number) => void
  updateLyricIndex: (index: number) => void
  updatePitch: (frequency: number | null) => void
  recordFrame: (isCorrect: boolean) => void
  setTranspose: (semitones: number) => void
  setTolerance: (cents: number) => void
  reset: () => void
}
```

---

## Type Exports

```typescript
// packages/engine/src/types.ts
export interface LyricsLine { t: number; l: string }
export interface ScoreFrame { ... }
export interface FinalScore { ... }
export interface PitchDetectionResult { ... }

// Re-export from index
export * from './types'
```
