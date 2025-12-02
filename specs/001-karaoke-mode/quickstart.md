# Quickstart: Karaoke Mode Validation

**Feature**: `001-karaoke-mode`
**Created**: 2024-12-02

---

## Purpose

This document outlines key validation scenarios to verify karaoke mode functionality. Use these for both manual QA and automated end-to-end testing.

---

## Prerequisites

1. Development server running (`pnpm dev`)
2. Test user account created
3. At least one published song with lyrics in database
4. Microphone available (for pitch detection tests)
5. Chrome browser (recommended for Web Audio API testing)

---

## Validation Scenarios

### Scenario 1: Basic Karaoke Session

**Goal**: Verify core playback and lyrics synchronization

**Steps**:
1. Navigate to a song detail page (`/songs/{id}`)
2. Click "Sing" or "Karaoke" button
3. Observe: Page transitions to karaoke mode
4. Observe: Audio starts playing
5. Observe: Lyrics panel displays all lines
6. Wait 10 seconds
7. Observe: Current lyric line is highlighted
8. Observe: Panel auto-scrolls to keep current line visible
9. Click pause button
10. Observe: Audio and highlighting stop
11. Click play button
12. Observe: Resumes from correct position

**Expected Result**: ✅ Audio and lyrics stay synchronized throughout playback

---

### Scenario 2: Seek Functionality

**Goal**: Verify seeking updates lyrics correctly

**Steps**:
1. Start a karaoke session
2. Wait for audio to reach 00:30
3. Drag progress bar to 01:00
4. Observe: Audio jumps to 01:00
5. Observe: Correct lyric for 01:00 is highlighted
6. Drag progress bar back to 00:15
7. Observe: Earlier lyrics are re-highlighted appropriately

**Expected Result**: ✅ Seeking forward/backward updates lyrics immediately

---

### Scenario 3: Microphone Permission Grant

**Goal**: Verify pitch detection activates with permission

**Steps**:
1. Start a karaoke session (first time)
2. Observe: Browser requests microphone permission
3. Click "Allow"
4. Observe: Pitch visualizer becomes active
5. Make a sound (hum or sing)
6. Observe: Pitch indicator responds to your voice

**Expected Result**: ✅ Pitch visualizer shows real-time pitch feedback

---

### Scenario 4: Microphone Permission Denied

**Goal**: Verify graceful degradation

**Steps**:
1. Clear site permissions for microphone
2. Start a karaoke session
3. When prompted, click "Block" or "Deny"
4. Observe: Karaoke continues playing
5. Observe: Message shows "Pitch feedback unavailable"
6. Complete the song
7. Observe: Score is based on completion only

**Expected Result**: ✅ Karaoke works without microphone, score shown without pitch data

---

### Scenario 5: Score Submission (Authenticated)

**Goal**: Verify score saves for logged-in users

**Steps**:
1. Log in as test user
2. Start and complete a karaoke session with microphone enabled
3. Observe: Final score summary appears
4. Observe: Score shows pitch accuracy and duration
5. Navigate to profile/history
6. Observe: Score appears in history
7. Return to same song
8. Observe: "Personal Best" badge if applicable

**Expected Result**: ✅ Score is persisted and retrievable

---

### Scenario 6: Keyboard Controls

**Goal**: Verify keyboard shortcuts work

**Steps**:
1. Start a karaoke session
2. Press `Spacebar`
3. Observe: Playback pauses
4. Press `Spacebar` again
5. Observe: Playback resumes
6. Press `Right Arrow`
7. Observe: Audio seeks forward 5 seconds
8. Press `Left Arrow`
9. Observe: Audio seeks back 5 seconds

**Expected Result**: ✅ All keyboard shortcuts function correctly

---

### Scenario 7: Khmer Lyrics Display

**Goal**: Verify Khmer text renders correctly

**Steps**:
1. Navigate to a song with Khmer lyrics
2. Start karaoke session
3. Observe: Khmer text displays without garbled characters
4. Observe: Text is readable size (not too small)
5. Observe: Line breaks occur at appropriate points

**Expected Result**: ✅ Khmer script renders beautifully with proper font

---

### Scenario 8: Mobile Responsiveness

**Goal**: Verify karaoke works on mobile devices

**Steps**:
1. Open karaoke page on mobile device (or emulator)
2. Tap to start (required for audio context on mobile)
3. Observe: UI fits mobile screen
4. Observe: Controls are touch-friendly
5. Complete a session

**Expected Result**: ✅ Full functionality on mobile browsers

---

### Scenario 9: Session Interruption

**Goal**: Verify handling of unexpected interruptions

**Steps**:
1. Start a karaoke session
2. Switch to another browser tab
3. Wait 10 seconds
4. Return to karaoke tab
5. Observe: Session state (audio may pause based on browser)
6. Resume if needed

**Expected Result**: ✅ Session can be resumed or provides clear indication of state

---

### Scenario 10: Tolerance Adjustment

**Goal**: Verify pitch tolerance setting affects scoring

**Steps**:
1. Start karaoke with default tolerance (80 cents)
2. Sing deliberately off-pitch
3. Note the score
4. Restart with higher tolerance (120 cents)
5. Sing similarly off-pitch
6. Compare scores

**Expected Result**: ✅ Higher tolerance results in better score for same performance

---

## Automated Test Commands

```bash
# Run unit tests for karaoke engine
pnpm --filter @karaoke/engine test

# Run integration tests
pnpm --filter @karaoke/trpc test

# Run e2e tests (requires running dev server)
pnpm --filter web e2e
```

---

## Known Limitations

1. **Safari Audio**: Requires user tap to start AudioContext
2. **Firefox Pitch**: Slightly different pitch detection behavior
3. **iOS Chrome**: Uses Safari's WebKit, same limitations apply
4. **Background Tab**: Most browsers pause audio when tab is hidden

---

## Issue Reporting

If a scenario fails, document:
1. Browser and version
2. Device type
3. Steps to reproduce
4. Expected vs actual behavior
5. Console errors (if any)
6. Screenshot or video
