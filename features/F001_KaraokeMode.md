# F001: Karaoke Mode

## Problem Statement

Users want to sing along to their favorite songs with visual feedback and scoring. Without real-time lyrics synchronization and pitch detection, the karaoke experience feels disconnected and unengaging.

## Solution

Build a full-featured karaoke mode that:
- Synchronizes lyrics with audio playback in real-time
- Captures microphone input for pitch detection
- Provides real-time visual feedback on singing accuracy
- Calculates and displays a final score

## User Stories

### US1: Start Singing
As a user, I want to start a karaoke session so that I can sing along with the song.

**Acceptance Criteria:**
- [ ] "Start Singing" button is prominently displayed on song page
- [ ] Clicking button requests microphone permission
- [ ] Audio begins playing after permission granted
- [ ] Error message shown if permission denied

### US2: Follow Lyrics
As a user, I want to see the current lyrics highlighted so that I know which line to sing.

**Acceptance Criteria:**
- [ ] Current lyric line is visually highlighted
- [ ] Highlight advances in sync with audio (±50ms tolerance)
- [ ] Khmer text renders correctly with proper fonts
- [ ] Lyrics panel auto-scrolls to keep current line visible

### US3: See Pitch Feedback
As a user, I want real-time feedback on my pitch so that I can improve my singing.

**Acceptance Criteria:**
- [ ] Detected pitch displayed in Hz
- [ ] Visual indicator shows if pitch is on/off target
- [ ] Feedback updates smoothly (no jarring jumps)
- [ ] Works with various voice types (male/female)

### US4: Adjust Difficulty
As a user, I want to adjust transpose and tolerance so that the song fits my voice range.

**Acceptance Criteria:**
- [ ] Transpose slider: -12 to +12 semitones
- [ ] Tolerance slider: 10 to 200 cents
- [ ] Changes apply immediately without restart
- [ ] Settings remembered for the session

### US5: View Final Score
As a user, I want to see my final score after singing so that I can track my progress.

**Acceptance Criteria:**
- [ ] Score displayed as 0-100 at end of song
- [ ] Breakdown shown: pitch accuracy %, duration coverage %
- [ ] Option to save score (if logged in)
- [ ] Option to sing again

## Implementation

### Components

| Component | Type | Description |
|-----------|------|-------------|
| KaraokeMode | Client | Main container, orchestrates all sub-components |
| AudioPlayer | Client | HTML5 audio with controls |
| LyricsPanel | Client | Displays lyrics, highlights current line |
| PitchVisualizer | Client | Shows detected pitch vs reference |
| ScoreDisplay | Client | Real-time and final score display |
| KaraokeControls | Client | Start/stop, transpose, tolerance sliders |

### State Management

```typescript
// stores/karaoke-store.ts
interface KaraokeState {
  isPlaying: boolean
  isSinging: boolean
  currentTime: number
  currentLyricIndex: number
  detectedFrequency: number | null
  transpose: number
  toleranceCents: number
  framesCorrect: number
  framesTotal: number
}
```

### Karaoke Engine Integration

```typescript
// hooks/useKaraoke.ts
export function useKaraoke(song: SongDTO) {
  const audioRef = useRef<HTMLAudioElement>(null)
  const engineRef = useRef<KaraokeEngine | null>(null)

  const startSinging = async () => {
    const stream = await navigator.mediaDevices.getUserMedia({ audio: true })
    engineRef.current = new KaraokeEngine({
      audioElement: audioRef.current!,
      lyrics: song.lyricsJson,
      micStream: stream,
      onScoreUpdate: (score) => setScore(score),
      onLyricChange: (index) => setCurrentIndex(index),
    })
    engineRef.current.start()
  }

  // ... rest of hook
}
```

### API Integration

- `song.get`: Fetch song with lyricsJson
- `score.submit`: Save final score (authenticated)
- `score.history`: Get user's score history for song

## Metrics

### Success Metrics

| Metric | Target | Measurement |
|--------|--------|-------------|
| Lyrics sync accuracy | ≤ 50ms drift | Test harness |
| Pitch detection latency | < 100ms | Performance monitoring |
| Session completion rate | > 60% | Analytics |
| Score submission rate | > 30% (logged in users) | Analytics |

### Monitoring

- Error tracking: Sentry for getUserMedia failures
- Performance: Web Vitals for karaoke page
- Usage: Track sessions started, completed, scores saved

## Dependencies

- Decisions: D001_KaraokeEngine, D008_PitchDetection
- Packages: @karaoke/karaoke-engine
- Browser APIs: Web Audio API, getUserMedia

## Related

- Features: F003_Recording, F004_Scoring
- Decisions: D001_KaraokeEngine, D008_PitchDetection
- Implementation: I004_KaraokeEngine
