# Implementation Plan: Karaoke Mode

**Branch**: `001-karaoke-mode`
**Created**: 2024-12-02
**Spec**: `specs/001-karaoke-mode/spec.md`

---

## Summary

Implement real-time karaoke functionality with lyrics synchronization, pitch detection using the YIN algorithm, and performance scoring. The implementation uses Web Audio API for client-side audio processing to achieve zero-latency feedback.

---

## Technical Context

| Aspect | Decision |
|--------|----------|
| Language | TypeScript 5.x |
| Runtime | Browser (Chrome, Firefox, Safari, Edge) |
| Framework | Next.js 16 with App Router |
| Audio API | Web Audio API (AudioContext, AnalyserNode) |
| State Management | Zustand |
| Styling | Tailwind CSS + shadcn/ui |
| Testing | Vitest + React Testing Library |

---

## Constitution Check

### Phase -1: Pre-Implementation Gates

#### Simplicity Gate (Article V)
- [x] Implementation uses existing packages only (`@karaoke/engine`, `@karaoke/utils`)
- [x] No new packages required beyond approved list
- [x] Components consolidated into `apps/web/components/karaoke/`

#### Anti-Abstraction Gate (Article VI)
- [x] Using Web Audio API directly, no abstraction layer
- [x] React hooks follow standard patterns
- [x] Zustand store uses direct state management

#### Integration-First Gate (Article VII)
- [x] Contract tests defined for score submission API
- [x] Audio processing tested with real audio files
- [ ] Browser compatibility tests with actual browsers

---

## Project Structure

### Specification Directory
```
specs/001-karaoke-mode/
├── spec.md                    # This file
├── plan.md                    # Implementation plan
├── data-model.md             # Data structures
├── contracts/
│   └── score-api.md          # Score submission contract
├── quickstart.md             # Validation scenarios
└── tasks.md                  # Generated task list
```

### Source Code Structure
```
packages/engine/
├── src/
│   ├── karaoke-engine.ts     # Main orchestrator class
│   ├── pitch-detector.ts     # YIN algorithm implementation
│   ├── lyrics-sync.ts        # Time-based lyrics synchronization
│   ├── score-calculator.ts   # Frame-based scoring
│   └── index.ts              # Public exports
├── tests/
│   ├── pitch-detector.test.ts
│   ├── lyrics-sync.test.ts
│   └── score-calculator.test.ts
└── package.json

apps/web/
├── app/(public)/songs/[id]/
│   └── karaoke/
│       └── page.tsx          # Karaoke mode page
├── components/karaoke/
│   ├── KaraokeMode.tsx       # Main karaoke component
│   ├── LyricsPanel.tsx       # Lyrics display
│   ├── PitchVisualizer.tsx   # Pitch feedback UI
│   ├── ScoreDisplay.tsx      # Real-time and final score
│   └── KaraokeControls.tsx   # Play/pause/seek controls
├── hooks/
│   ├── useKaraoke.ts         # Orchestrates karaoke session
│   ├── usePitchDetection.ts  # Microphone + pitch detection
│   └── useLyricsSync.ts      # Audio-lyrics synchronization
└── stores/
    └── karaoke-store.ts      # Zustand store for session state
```

---

## Technical Design

### Architecture Overview

```
┌─────────────────────────────────────────────────────────────────────┐
│                         KARAOKE MODE ARCHITECTURE                    │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  ┌──────────────┐     ┌───────────────────────────────────────────┐│
│  │   Song Data  │────▶│              KaraokeEngine                 ││
│  │  (API fetch) │     │  ┌────────────┐  ┌────────────────────┐  ││
│  └──────────────┘     │  │LyricsSync  │  │   PitchDetector    │  ││
│                       │  │            │  │   (YIN Algorithm)   │  ││
│  ┌──────────────┐     │  │ Binary     │  │                    │  ││
│  │ HTMLAudioEl  │────▶│  │ Search     │  │  ┌──────────────┐  │  ││
│  │ (Playback)   │     │  │ O(log n)   │  │  │ AudioContext │  │  ││
│  └──────────────┘     │  └─────┬──────┘  │  │ AnalyserNode │  │  ││
│                       │        │         │  └───────┬──────┘  │  ││
│  ┌──────────────┐     │        ▼         │          │         │  ││
│  │  Microphone  │────▶│  currentIndex    │          ▼         │  ││
│  │  (getUserMe- │     │                  │   detectedFreq     │  ││
│  │   dia())     │     │  ┌───────────────┴──────────┐         │  ││
│  └──────────────┘     │  │    ScoreCalculator       │         │  ││
│                       │  │    (frame comparison)    │         │  ││
│                       │  └───────────┬──────────────┘         │  ││
│                       └──────────────┼────────────────────────┘ │
│                                      │                           │
│                                      ▼                           │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │                    Zustand Store                           │  │
│  │  { isPlaying, currentTime, currentIndex, score, ... }     │  │
│  └───────────────────────────────────────────────────────────┘  │
│                                      │                           │
│                                      ▼                           │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │                    React Components                        │  │
│  │  KaraokeMode → LyricsPanel + PitchVisualizer + Controls   │  │
│  └───────────────────────────────────────────────────────────┘  │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

### Core Classes

#### KaraokeEngine
Orchestrates all karaoke functionality:
- Initializes audio playback
- Connects LyricsSync, PitchDetector, ScoreCalculator
- Manages session lifecycle (start, pause, resume, stop)
- Exposes state via callback or observable pattern

#### PitchDetector
Implements YIN algorithm for monophonic pitch detection:
- Input: Float32Array audio buffer (2048 samples @ 44100Hz)
- Output: Frequency in Hz or null (no pitch detected)
- Threshold: 0.15 (configurable)
- Latency budget: ≤ 15ms

#### LyricsSync
Synchronizes lyrics display with audio playback:
- Input: Lyrics array sorted by timestamp
- Uses binary search O(log n) to find current line
- Updates via requestAnimationFrame for smooth 60fps
- Handles seeking (forward/backward jump)

#### ScoreCalculator
Calculates performance score from pitch data:
- Compares detected pitch to expected pitch (if available)
- Falls back to pitch stability scoring
- Tracks: frames correct, frames total, duration coverage
- Final score: 0-100 based on weighted formula

---

## File Creation Order

### Phase 1: Engine Package (Tests First)

1. `packages/engine/tests/pitch-detector.test.ts` - Test YIN algorithm accuracy
2. `packages/engine/src/pitch-detector.ts` - Implement YIN algorithm
3. `packages/engine/tests/lyrics-sync.test.ts` - Test sync precision
4. `packages/engine/src/lyrics-sync.ts` - Implement lyrics synchronization
5. `packages/engine/tests/score-calculator.test.ts` - Test scoring formula
6. `packages/engine/src/score-calculator.ts` - Implement scoring
7. `packages/engine/src/karaoke-engine.ts` - Orchestrator class
8. `packages/engine/src/index.ts` - Public exports

### Phase 2: Web App Integration

9. `apps/web/stores/karaoke-store.ts` - Zustand store
10. `apps/web/hooks/usePitchDetection.ts` - Microphone + detection hook
11. `apps/web/hooks/useLyricsSync.ts` - Lyrics sync hook
12. `apps/web/hooks/useKaraoke.ts` - Main orchestration hook
13. `apps/web/components/karaoke/LyricsPanel.tsx` - Lyrics display
14. `apps/web/components/karaoke/PitchVisualizer.tsx` - Pitch feedback
15. `apps/web/components/karaoke/ScoreDisplay.tsx` - Score UI
16. `apps/web/components/karaoke/KaraokeControls.tsx` - Playback controls
17. `apps/web/components/karaoke/KaraokeMode.tsx` - Main component
18. `apps/web/app/(public)/songs/[id]/karaoke/page.tsx` - Route page

### Phase 3: API Integration

19. `packages/trpc/routers/score.ts` - Score submission endpoint
20. Integration tests for score persistence

---

## Complexity Tracking

| Article | Deviation | Justification | Date |
|---------|-----------|---------------|------|
| None | N/A | N/A | - |

---

## Supporting Documents

- `data-model.md` - LyricsLine, ScoreFrame, KaraokeSession structures
- `contracts/score-api.md` - tRPC score.submit contract
- `quickstart.md` - Manual validation scenarios
- `tasks.md` - Generated task list for implementation

---

## Risk Mitigation

| Risk | Mitigation |
|------|------------|
| Safari AudioContext restrictions | User gesture required; show "Tap to Start" |
| High CPU on mobile | Reduce pitch detection frequency if needed |
| Microphone permission denied | Graceful degradation to duration-only scoring |
| Large lyrics causing lag | Virtual scrolling for > 100 lines |
