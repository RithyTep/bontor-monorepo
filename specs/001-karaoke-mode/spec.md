# Feature Specification: Karaoke Mode

**Branch**: `001-karaoke-mode`
**Created**: 2024-12-02
**Status**: Ready for Planning

---

## Summary

Enable users to sing along to songs with real-time lyrics highlighting, pitch visualization, and performance scoring. This is the core feature that differentiates the platform from simple music streaming.

---

## User Scenarios & Testing

### US1: Play Song with Lyrics (P1 - MVP Critical)

**Context**: Users need to see lyrics synchronized with audio playback to sing along effectively.

**Story**: As a user, I want to play a song and see lyrics highlighted in real-time so that I can sing along without memorizing the words.

**Acceptance Scenarios**:

```gherkin
Scenario: Start karaoke session
  Given I am on a song detail page
  When I click the "Sing" button
  Then the audio should start playing
  And the lyrics panel should display all lyrics
  And the current lyric line should be highlighted
  And highlighting should advance as audio plays

Scenario: Lyrics stay synchronized during playback
  Given a karaoke session is active
  When the audio reaches timestamp 00:30
  Then the lyric line for 00:30 should be highlighted
  And previously sung lines should appear dimmed
  And upcoming lines should be visible but not highlighted

Scenario: Handle songs with instrumental sections
  Given a song has an instrumental break (no lyrics for 20+ seconds)
  When the audio reaches the instrumental section
  Then a visual indicator should show "♪ Instrumental ♪"
  And the next lyric should be visible as upcoming
```

**Edge Cases**:
- Song with very fast lyrics (< 1 second per line)
- Song with very long lines that need scrolling
- Songs mixing Khmer and English lyrics
- Browser tab loses focus during playback

---

### US2: Control Playback (P1 - MVP Critical)

**Context**: Users need standard media controls to manage their singing session.

**Story**: As a user, I want to pause, resume, and seek within the song so that I can practice specific sections.

**Acceptance Scenarios**:

```gherkin
Scenario: Pause and resume playback
  Given a karaoke session is playing
  When I click the pause button
  Then audio should pause
  And lyrics highlighting should pause
  When I click play
  Then audio should resume from the paused position
  And lyrics should continue from the correct line

Scenario: Seek to specific position
  Given a karaoke session is active
  When I drag the progress bar to 01:30
  Then audio should jump to 01:30
  And the correct lyric for that timestamp should be highlighted
  And previous lyrics should show as already sung

Scenario: Use keyboard shortcuts
  Given a karaoke session is active
  When I press Spacebar
  Then playback should toggle (play/pause)
  When I press Left Arrow
  Then audio should seek back 5 seconds
  When I press Right Arrow
  Then audio should seek forward 5 seconds
```

**Edge Cases**:
- Seeking beyond song duration
- Rapid pause/play toggling
- Seeking while audio is still loading

---

### US3: See Pitch Feedback (P2 - Core Experience)

**Context**: Users benefit from visual feedback showing if they're singing the correct pitch.

**Story**: As a user, I want to see real-time visualization of my voice pitch compared to the expected pitch so that I can improve my singing.

**Acceptance Scenarios**:

```gherkin
Scenario: Enable microphone for pitch detection
  Given I am starting a karaoke session
  When the system requests microphone permission
  And I grant permission
  Then the pitch visualizer should activate
  And I should see my detected pitch displayed

Scenario: Display pitch comparison
  Given microphone is enabled
  And I am singing
  When my voice is detected
  Then a visual indicator should show my pitch
  And the expected pitch should be displayed (if available)
  And color coding should indicate if I'm on pitch (green), sharp (blue), or flat (red)

Scenario: Handle microphone denied
  Given I start a karaoke session
  When I deny microphone permission
  Then karaoke should continue without pitch feedback
  And a message should indicate "Pitch feedback unavailable"
  And the score should be based on duration coverage only
```

**Edge Cases**:
- Background noise triggering false pitch detection
- User stops singing (silence)
- Microphone disconnected mid-session
- Very high or very low voices outside typical range

---

### US4: Adjust Key/Transpose (P2 - Core Experience)

**Context**: Different users have different vocal ranges; the same song may be too high or low for some singers.

**Story**: As a user, I want to adjust the pitch tolerance or understand that I can sing in a different octave so that I can score well regardless of my vocal range.

**Acceptance Scenarios**:

```gherkin
Scenario: Adjust pitch tolerance
  Given I am in karaoke mode
  When I adjust the "Tolerance" slider from 80 to 120 cents
  Then pitch matching should become more forgiving
  And my score should reflect the new tolerance

Scenario: Display transpose guidance
  Given the song is in a key uncomfortable for me
  When I open settings
  Then I should see guidance that octave shifts are acceptable
  And the scoring should accept pitches ±1 octave from target

[NEEDS CLARIFICATION: Should we support actual audio transposition, or just adjust pitch matching tolerance?]
```

**Edge Cases**:
- User sets tolerance to maximum (very forgiving)
- User sets tolerance to minimum (very strict)

---

### US5: View Performance Score (P1 - MVP Critical)

**Context**: Scoring motivates users and gamifies the karaoke experience.

**Story**: As a user, I want to see my score during and after singing so that I know how well I performed.

**Acceptance Scenarios**:

```gherkin
Scenario: See real-time score
  Given I am singing with microphone enabled
  When I sing along to the lyrics
  Then I should see a running accuracy percentage
  And the score should update in real-time

Scenario: View final score summary
  Given I have completed a song
  When the song ends
  Then I should see a score summary showing:
    - Total score (0-100)
    - Pitch accuracy percentage
    - Duration coverage percentage
  And I should see my personal best for this song (if logged in)

Scenario: Score without microphone
  Given microphone permission was denied
  When I complete the song
  Then score should be based on playback completion only
  And a note should indicate "No pitch data available"
```

**Edge Cases**:
- User stops singing halfway through
- User only sings every other line
- Network disconnection prevents score save

---

## Requirements

### Functional Requirements

| ID | Requirement | User Story |
|----|-------------|------------|
| FR1 | Play audio with time tracking | US1, US2 |
| FR2 | Display lyrics synchronized to audio within ±100ms | US1 |
| FR3 | Highlight current lyric line and auto-scroll | US1 |
| FR4 | Support play, pause, seek controls | US2 |
| FR5 | Keyboard shortcuts for playback control | US2 |
| FR6 | Request microphone permission | US3 |
| FR7 | Detect pitch using YIN algorithm | US3 |
| FR8 | Display pitch visualization | US3 |
| FR9 | Calculate and display real-time score | US5 |
| FR10 | Show final score summary on song completion | US5 |
| FR11 | Adjust pitch tolerance setting | US4 |
| FR12 | Handle graceful degradation without microphone | US3, US5 |

### Non-Functional Requirements

| ID | Requirement | Metric |
|----|-------------|--------|
| NFR1 | Audio-to-visual sync latency | ≤ 50ms |
| NFR2 | Pitch detection latency | ≤ 50ms |
| NFR3 | UI responsiveness during playback | 60 FPS |
| NFR4 | Initial load time for karaoke mode | ≤ 2 seconds |
| NFR5 | CPU usage on mid-range mobile | ≤ 30% |
| NFR6 | Works on Chrome, Firefox, Safari, Edge | Latest 2 versions |
| NFR7 | Works on iOS Safari and Chrome Android | iOS 15+, Android 10+ |

### Key Entities

| Entity | Description |
|--------|-------------|
| Song | Audio file, lyrics data, metadata |
| LyricsLine | Timestamp (seconds) + text content |
| KaraokeSession | Active session with audio element, sync state |
| ScoreFrame | Individual pitch comparison result |
| FinalScore | Aggregated score with breakdown |

---

## Success Criteria

| Metric | Target | Measurement Method |
|--------|--------|-------------------|
| Karaoke session completion rate | ≥ 60% | Analytics: started vs completed |
| Average session duration | ≥ 2 minutes | Analytics: time in karaoke mode |
| User return rate (sing again) | ≥ 40% | Analytics: repeat karaoke sessions |
| Pitch detection accuracy | ≥ 95% within ±50 cents | Automated test suite |
| Lyrics sync accuracy | ≥ 99% within ±100ms | Manual QA + automated tests |

---

## Requirement Completeness Checklist

- [x] All user stories have acceptance scenarios in Given-When-Then format
- [x] Each user story is independently testable
- [x] Success criteria are measurable
- [x] Edge cases are documented
- [x] Non-functional requirements have specific metrics
- [ ] All `[NEEDS CLARIFICATION]` items resolved

---

## Open Questions

1. **Audio Transposition**: Should we support actual audio key change, or just adjust pitch tolerance for scoring? (Current: tolerance only)

2. **Reference Pitch**: Do we have reference pitch data for songs, or is scoring purely based on pitch stability and duration? (Current: duration + stability)
