# D001: Karaoke Engine Architecture

## Status
- [ ] Proposed
- [x] Approved
- [ ] Superseded by D###

## Context

The Karaoke Cambodia application requires a real-time karaoke experience including:
- Audio playback with synchronized lyrics
- Microphone input capture
- Pitch detection for user's voice
- Real-time scoring feedback
- Final score calculation

The key architectural decision is where to process the audio and calculate scores: client-side, server-side, or a hybrid approach.

## Alternatives Considered

### Option A: Server-Side Scoring

**Description:** Stream mic audio to server, process pitch detection and scoring on backend, return results to client.

**Pros:**
- Consistent scoring across devices
- Harder to cheat/manipulate scores
- Can use heavier ML models for better accuracy
- Easier to update scoring algorithm

**Cons:**
- High latency (network round-trip)
- Poor real-time feedback UX
- High server costs for audio processing
- Requires WebSocket or streaming infrastructure
- Complex infrastructure for audio streaming

### Option B: Client-Side Scoring

**Description:** All pitch detection and scoring happens in the browser using Web Audio API.

**Pros:**
- Zero network latency for feedback
- Real-time pitch visualization possible
- Lower server costs
- Works offline (future PWA)
- Simpler backend architecture

**Cons:**
- Device-dependent performance
- Scoring algorithm exposed to client
- Potential for manipulation (not critical for MVP)
- Need fallback for unsupported browsers

### Option C: Hybrid Approach

**Description:** Client-side real-time feedback with optional server-side score verification.

**Pros:**
- Best of both worlds
- Real-time UX with verification option
- Flexibility for future leaderboards

**Cons:**
- Most complex to implement
- Requires recording upload
- Delayed final score if verified

## Decision

**Client-side scoring with optional server verification (future)**

For MVP, implement full client-side pitch detection and scoring using Web Audio API. Structure the code to allow future server-side verification for competitive features.

## Rationale

1. **User Experience Priority:** Real-time feedback is critical for karaoke enjoyment. Latency would destroy the singing experience.

2. **MVP Simplicity:** Server-side audio processing adds significant infrastructure complexity that isn't justified for MVP.

3. **Cost Efficiency:** Audio processing servers are expensive. Client-side processing leverages user's device.

4. **Modern Browser Support:** Web Audio API is well-supported in target browsers (Chrome, Safari, Firefox).

5. **Future-Proof:** The hybrid approach can be added later for leaderboard features without changing the core client experience.

## Impact

### Technical
- Frontend will include karaoke-engine package with pitch detection
- Web Audio API required (browser compatibility consideration)
- No real-time server infrastructure needed for MVP
- Score data saved to DB for history (post-session)

### Product
- Instant feedback enables engaging UX
- Scores may vary slightly between devices (acceptable for MVP)
- Leaderboard integrity deferred to post-MVP

### Resources
- Frontend team owns karaoke engine development
- No specialized audio processing backend needed

## Implementation Notes

- Use YIN algorithm for pitch detection (see D008)
- Implement as separate `@karaoke/karaoke-engine` package
- Use `requestAnimationFrame` for sync loops
- Support transpose control for user comfort
- Allow tolerance adjustment for difficulty

## Related

- Features: F001_KaraokeMode, F004_Scoring
- Decisions: D008_PitchDetection
- Implementation: I004_KaraokeEngine
- Proposals: P001_AudioProcessingPipeline
