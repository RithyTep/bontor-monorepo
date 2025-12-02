# Tasks: Karaoke Mode Implementation

**Feature**: `001-karaoke-mode`
**Generated**: 2024-12-02
**Source**: `plan.md`, `data-model.md`, `contracts/score-api.md`

---

## Task Format

`[ID] [P?] [Story] Description`

- **[P]** = Can run in parallel (different files, no dependencies)
- **[Story]** = Traces to user story (US1, US2, etc.)

---

## Phase 1: Setup

- [ ] `[1.1]` Initialize `packages/engine` package with TypeScript config
- [ ] `[1.2]` Add engine package to workspace dependencies in `apps/web`
- [ ] `[1.3]` Configure Vitest for engine package testing

**Checkpoint**: Engine package scaffolded and can import in web app

---

## Phase 2: Foundational (Blocks All User Stories)

### Pitch Detection

- [ ] `[2.1] [P]` Write tests for YIN algorithm (`pitch-detector.test.ts`)
- [ ] `[2.2]` Implement YIN pitch detection algorithm (`pitch-detector.ts`)
- [ ] `[2.3]` Verify pitch detection accuracy ≥95% with test audio samples

### Lyrics Synchronization

- [ ] `[2.4] [P]` Write tests for lyrics sync binary search (`lyrics-sync.test.ts`)
- [ ] `[2.5]` Implement LyricsSync class with binary search (`lyrics-sync.ts`)
- [ ] `[2.6]` Verify sync precision ≤50ms with test data

### Score Calculator

- [ ] `[2.7] [P]` Write tests for score calculation formula (`score-calculator.test.ts`)
- [ ] `[2.8]` Implement ScoreCalculator class (`score-calculator.ts`)

### Engine Orchestrator

- [ ] `[2.9]` Implement KaraokeEngine orchestrator class (`karaoke-engine.ts`)
- [ ] `[2.10]` Export all public types and classes (`index.ts`)

**Checkpoint**: `@karaoke/engine` package complete, all tests passing

---

## Phase 3: User Story 1 - Play Song with Lyrics (P1)

### State Management

- [ ] `[3.1]` Create Zustand store with karaoke session state (`karaoke-store.ts`)

### Hooks

- [ ] `[3.2] [P]` Implement lyrics sync hook (`useLyricsSync.ts`)
- [ ] `[3.3] [P]` Implement audio playback orchestration hook (`useKaraoke.ts` - basic)

### Components

- [ ] `[3.4]` Implement LyricsPanel component with highlighting (`LyricsPanel.tsx`)
- [ ] `[3.5]` Implement basic KaraokeMode wrapper component (`KaraokeMode.tsx` - v1)

### Page Route

- [ ] `[3.6]` Create karaoke page route (`app/(public)/songs/[id]/karaoke/page.tsx`)
- [ ] `[3.7]` Fetch song data with lyrics on page load

### Testing

- [ ] `[3.8]` Write component tests for LyricsPanel
- [ ] `[3.9]` Manual QA: Verify lyrics sync with actual audio

**Checkpoint**: US1 complete - Users can play song with synchronized lyrics

---

## Phase 4: User Story 2 - Control Playback (P1)

### Components

- [ ] `[4.1]` Implement KaraokeControls with play/pause/seek (`KaraokeControls.tsx`)
- [ ] `[4.2]` Add progress bar with drag-to-seek functionality
- [ ] `[4.3]` Integrate controls into KaraokeMode component

### Keyboard Support

- [ ] `[4.4]` Add keyboard event listeners (Space, Arrow keys)
- [ ] `[4.5]` Handle focus management for keyboard controls

### Testing

- [ ] `[4.6]` Write tests for keyboard shortcuts
- [ ] `[4.7]` Manual QA: Verify seek updates lyrics correctly

**Checkpoint**: US2 complete - Users can pause, resume, seek with keyboard shortcuts

---

## Phase 5: User Story 3 - Pitch Feedback (P2)

### Hooks

- [ ] `[5.1]` Implement usePitchDetection hook with getUserMedia (`usePitchDetection.ts`)
- [ ] `[5.2]` Handle microphone permission states (granted, denied, prompt)

### Components

- [ ] `[5.3]` Implement PitchVisualizer component (`PitchVisualizer.tsx`)
- [ ] `[5.4]` Show pitch indicator (current pitch vs target)
- [ ] `[5.5]` Implement "no microphone" fallback UI

### Integration

- [ ] `[5.6]` Connect pitch detection to KaraokeMode
- [ ] `[5.7]` Update Zustand store with pitch data

### Testing

- [ ] `[5.8]` Write tests for permission handling
- [ ] `[5.9]` Manual QA: Verify pitch visualization responsiveness

**Checkpoint**: US3 complete - Users see real-time pitch feedback

---

## Phase 6: User Story 4 - Tolerance Adjustment (P2)

### Components

- [ ] `[6.1]` Add tolerance slider to settings UI
- [ ] `[6.2]` Display current tolerance value

### Logic

- [ ] `[6.3]` Connect tolerance setting to ScoreCalculator
- [ ] `[6.4]` Persist tolerance preference in localStorage

### Testing

- [ ] `[6.5]` Verify tolerance affects scoring accuracy

**Checkpoint**: US4 complete - Users can adjust pitch tolerance

---

## Phase 7: User Story 5 - Performance Score (P1)

### Components

- [ ] `[7.1]` Implement real-time ScoreDisplay component (`ScoreDisplay.tsx`)
- [ ] `[7.2]` Implement final score summary modal/panel
- [ ] `[7.3]` Show breakdown (pitch accuracy, duration coverage)

### API Integration

- [ ] `[7.4]` Implement score.submit tRPC procedure (`routers/score.ts`)
- [ ] `[7.5]` Implement score.history tRPC procedure
- [ ] `[7.6]` Implement score.best tRPC procedure
- [ ] `[7.7]` Implement score.leaderboard tRPC procedure

### Database

- [ ] `[7.8]` Add Score model to Prisma schema
- [ ] `[7.9]` Run migration for Score table
- [ ] `[7.10]` Add indexes for score queries

### Client Integration

- [ ] `[7.11]` Call score.submit on session completion
- [ ] `[7.12]` Display personal best badge
- [ ] `[7.13]` Show leaderboard preview

### Testing

- [ ] `[7.14] [P]` Write contract tests for score API
- [ ] `[7.15] [P]` Write component tests for ScoreDisplay
- [ ] `[7.16]` Manual QA: Complete session and verify score persistence

**Checkpoint**: US5 complete - Users see scores and can track progress

---

## Phase 8: Polish

### Performance

- [ ] `[8.1]` Implement virtual scrolling for long lyrics (>100 lines)
- [ ] `[8.2]` Optimize re-renders with React.memo
- [ ] `[8.3]` Profile and optimize pitch detection CPU usage

### UX Improvements

- [ ] `[8.4]` Add loading states for audio and lyrics
- [ ] `[8.5]` Add error boundaries for audio/mic failures
- [ ] `[8.6]` Implement "Tap to Start" for mobile Safari

### Accessibility

- [ ] `[8.7]` Add ARIA labels to controls
- [ ] `[8.8]` Ensure keyboard navigation works throughout
- [ ] `[8.9]` Test with screen reader

### Mobile

- [ ] `[8.10]` Test and fix responsive layout issues
- [ ] `[8.11]` Optimize touch targets for controls

### Cross-Browser

- [ ] `[8.12]` Test on Chrome, Firefox, Safari, Edge
- [ ] `[8.13]` Fix any browser-specific issues

**Checkpoint**: Feature polished and production-ready

---

## Parallel Task Groups

The following tasks can run simultaneously:

**Group A** (Engine tests - no dependencies):
- `[2.1]`, `[2.4]`, `[2.7]`

**Group B** (US3 hooks - after Phase 2):
- `[5.1]`, `[5.2]`

**Group C** (US5 API - after schema):
- `[7.4]`, `[7.5]`, `[7.6]`, `[7.7]`

**Group D** (Final tests - after implementation):
- `[7.14]`, `[7.15]`

---

## Dependencies Graph

```
Phase 1 (Setup)
    │
    ▼
Phase 2 (Foundational)
    │
    ├──────────────┬──────────────┐
    ▼              ▼              ▼
Phase 3 (US1)  Phase 4 (US2)  Phase 7 (US5)
    │              │              │
    └──────┬───────┘              │
           ▼                      │
    Phase 5 (US3)                 │
           │                      │
           ▼                      │
    Phase 6 (US4)                 │
           │                      │
           └──────────────────────┘
                      │
                      ▼
               Phase 8 (Polish)
```

---

## Estimated Effort

| Phase | Tasks | Parallel? | Estimate |
|-------|-------|-----------|----------|
| Setup | 3 | No | 0.5 days |
| Foundational | 10 | Partial | 2 days |
| US1 (Lyrics) | 9 | Partial | 1.5 days |
| US2 (Controls) | 7 | No | 1 day |
| US3 (Pitch) | 9 | Partial | 1.5 days |
| US4 (Tolerance) | 5 | No | 0.5 days |
| US5 (Score) | 16 | Partial | 2 days |
| Polish | 13 | Partial | 2 days |

**Total**: ~11 days (with parallelization: ~8 days)
